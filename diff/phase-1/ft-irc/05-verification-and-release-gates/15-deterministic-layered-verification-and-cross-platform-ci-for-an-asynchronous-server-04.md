## `test(app): 작은 송신 한도에서 상태 정리 검증`

diff --git a/.gitignore b/.gitignore
index 4e22b05..63de748 100644
--- a/.gitignore
+++ b/.gitignore
@@ -5,3 +5,4 @@ ircserv
 *.log
 tests/connection_test
 tests/server_lifetime_test
+tests/application_lifetime_test
diff --git a/Makefile b/Makefile
index 731b739..7846948 100644
--- a/Makefile
+++ b/Makefile
@@ -1,6 +1,7 @@
 NAME := irc-relay-server
 CONNECTION_TEST := tests/connection_test
 SERVER_LIFETIME_TEST := tests/server_lifetime_test
+APPLICATION_LIFETIME_TEST := tests/application_lifetime_test
 
 CXX ?= c++
 CXXFLAGS ?= -std=c++17 -Wall -Wextra -Werror -g
@@ -25,7 +26,7 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
 
-.PHONY: all clean connection-test fclean re test unit smoke
+.PHONY: all application-test clean connection-test fclean re test unit smoke
 
 all: $(NAME)
 
@@ -47,14 +48,22 @@ $(SERVER_LIFETIME_TEST): tests/server_lifetime_test.cpp src/Connection.cpp src/S
 unit: $(SERVER_LIFETIME_TEST)
 	./$(SERVER_LIFETIME_TEST)
 
-test: all connection-test unit
+APP_TEST_SRCS := $(filter-out src/main.cpp,$(SRCS))
+
+$(APPLICATION_LIFETIME_TEST): tests/application_lifetime_test.cpp $(APP_TEST_SRCS) src/IrcApplication.hpp src/ClientRegistry.hpp
+	$(CXX) $(CPPFLAGS) -Isrc $(CXXFLAGS) tests/application_lifetime_test.cpp $(APP_TEST_SRCS) -o $@
+
+application-test: $(APPLICATION_LIFETIME_TEST)
+	./$(APPLICATION_LIFETIME_TEST)
+
+test: all connection-test unit application-test
 	bash tests/irc_smoke.sh
 
 smoke: test
 
 clean:
-	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST) $(SERVER_LIFETIME_TEST)
-	rm -rf $(CONNECTION_TEST).dSYM $(SERVER_LIFETIME_TEST).dSYM
+	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST) $(SERVER_LIFETIME_TEST) $(APPLICATION_LIFETIME_TEST)
+	rm -rf $(CONNECTION_TEST).dSYM $(SERVER_LIFETIME_TEST).dSYM $(APPLICATION_LIFETIME_TEST).dSYM
 
 fclean: clean
 	rm -f $(NAME)
diff --git a/tests/application_lifetime_test.cpp b/tests/application_lifetime_test.cpp
new file mode 100644
index 0000000..118d15f
--- /dev/null
+++ b/tests/application_lifetime_test.cpp
@@ -0,0 +1,176 @@
+#include "EventManager.hpp"
+#include "IrcApplication.hpp"
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
+#include <sstream>
+#include <stdexcept>
+#include <string>
+#include <unordered_map>
+#include <utility>
+#include <vector>
+
+namespace {
+class CapturedStderr {
+public:
+    CapturedStderr() : previous_(std::cerr.rdbuf(stream_.rdbuf())) {}
+    ~CapturedStderr() { std::cerr.rdbuf(previous_); }
+    std::string str() const { return stream_.str(); }
+private:
+    std::ostringstream stream_;
+    std::streambuf* previous_;
+};
+
+class FakeEventManager : public irc::EventManager {
+public:
+    void addFd(int fd, irc::EventInterest interests) override { interests_[fd] = interests; }
+    void updateFd(int fd, irc::EventInterest interests) override {
+        if (interests_.find(fd) == interests_.end()) {
+            throw std::runtime_error("updated an unregistered descriptor");
+        }
+        interests_[fd] = interests;
+    }
+    void removeFd(int fd) override { interests_.erase(fd); }
+    std::vector<irc::Event> wait(int) override {
+        std::vector<irc::Event> ready;
+        ready.swap(events_);
+        return ready;
+    }
+    void queueReadable(int fd) {
+        irc::Event event;
+        event.fd = fd;
+        event.interests = irc::EventInterest::Read;
+        events_.push_back(event);
+    }
+    int clientFd(int listenFd) const {
+        for (std::unordered_map<int, irc::EventInterest>::const_iterator it = interests_.begin();
+             it != interests_.end(); ++it) {
+            if (it->first != listenFd) {
+                return it->first;
+            }
+        }
+        return -1;
+    }
+private:
+    std::unordered_map<int, irc::EventInterest> interests_;
+    std::vector<irc::Event> events_;
+};
+
+class ClientSocket {
+public:
+    explicit ClientSocket(std::uint16_t port) : fd_(::socket(AF_INET, SOCK_STREAM, 0)) {
+        if (fd_ == -1) {
+            throw std::runtime_error(std::string("socket: ") + std::strerror(errno));
+        }
+        sockaddr_in address;
+        std::memset(&address, 0, sizeof(address));
+        address.sin_family = AF_INET;
+        address.sin_port = htons(port);
+        address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
+        if (::connect(fd_, reinterpret_cast<sockaddr*>(&address), sizeof(address)) == -1) {
+            throw std::runtime_error(std::string("connect: ") + std::strerror(errno));
+        }
+    }
+    ~ClientSocket() { if (fd_ != -1) { ::close(fd_); } }
+    void sendRegistration() {
+        const std::string bytes = "NICK tiny\r\nUSER tiny 0 * :Tiny User\r\n";
+        std::size_t offset = 0;
+        while (offset < bytes.size()) {
+            const ssize_t count = ::send(fd_, bytes.data() + offset, bytes.size() - offset, 0);
+            if (count > 0) {
+                offset += static_cast<std::size_t>(count);
+            } else if (count == -1 && errno == EINTR) {
+                continue;
+            } else {
+                throw std::runtime_error(std::string("send: ") + std::strerror(errno));
+            }
+        }
+    }
+private:
+    int fd_;
+};
+
+void require(bool condition, const std::string& message) {
+    if (!condition) {
+        throw std::runtime_error(message);
+    }
+}
+
+void waitReadable(int fd) {
+    pollfd descriptor;
+    descriptor.fd = fd;
+    descriptor.events = POLLIN;
+    descriptor.revents = 0;
+    const int result = ::poll(&descriptor, 1, 1000);
+    if (result <= 0 || (descriptor.revents & POLLIN) == 0) {
+        throw std::runtime_error("accepted socket did not become readable");
+    }
+}
+
+void acceptUntil(irc::Server& server, FakeEventManager& events, std::size_t expectedCount) {
+    for (int attempt = 0; attempt < 10; ++attempt) {
+        waitReadable(server.listenFd());
+        events.queueReadable(server.listenFd());
+        server.pollOnce(0);
+        if (server.connectionCount() >= expectedCount) {
+            return;
+        }
+    }
+    throw std::runtime_error("accepted connection was not registered");
+}
+
+void registrationQueueFailureTest() {
+    CapturedStderr captured;
+    std::unique_ptr<FakeEventManager> ownedEvents(new FakeEventManager());
+    FakeEventManager* events = ownedEvents.get();
+    irc::Server::Config config;
+    config.bindAddress = "127.0.0.1";
+    config.port = 0;
+    config.eventTimeoutMs = 0;
+    config.maxPendingBytes = 1;
+    irc::Server server(config, std::move(ownedEvents));
+    RuntimeConfig runtime;
+    IrcApplication app(server, "", runtime);
+    server.setConnectHandler([&app](Connection& connection) { app.onConnect(connection); });
+    server.setLineHandler([&app](Connection& connection, const std::string& line) { app.onLine(connection, line); });
+    server.setDisconnectHandler([&app](Connection& connection, const std::string& reason) { app.onDisconnect(connection, reason); });
+    server.start();
+
+    ClientSocket client(server.port());
+    acceptUntil(server, *events, 1);
+    const int clientFd = events->clientFd(server.listenFd());
+    require(clientFd != -1, "accepted connection was not registered");
+    client.sendRegistration();
+    waitReadable(clientFd);
+    events->queueReadable(clientFd);
+    server.pollOnce(0);
+
+    require(server.connectionCount() == 0, "queue rejection left an application connection behind");
+    require(server.isRunning(), "application queue rejection stopped the server");
+    require(captured.str().find("event=client_registered") == std::string::npos,
+            "registration was recorded after the connection was removed");
+    server.setConnectHandler(Server::ConnectHandler());
+    server.setLineHandler(Server::LineHandler());
+    server.setDisconnectHandler(Server::DisconnectHandler());
+    server.stop();
+}
+} // namespace
+
+int main() {
+    try {
+        registrationQueueFailureTest();
+    } catch (const std::exception& exception) {
+        std::cerr << "application lifetime test failed: " << exception.what() << '\n';
+        return 1;
+    }
+    std::cout << "application lifetime test passed\n";
+    return 0;
+}


## `test(event): 160개 연결과 느린 수신자 처리 공정성 검증`

diff --git a/Makefile b/Makefile
index 7846948..bfe6f3b 100644
--- a/Makefile
+++ b/Makefile
@@ -26,7 +26,7 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
 
-.PHONY: all application-test clean connection-test fclean re test unit smoke
+.PHONY: all application-test clean connection-test event-test fclean re smoke test unit
 
 all: $(NAME)
 
@@ -58,6 +58,10 @@ application-test: $(APPLICATION_LIFETIME_TEST)
 
 test: all connection-test unit application-test
 	bash tests/irc_smoke.sh
+	$(MAKE) event-test
+
+event-test: all
+	PYTHONDONTWRITEBYTECODE=1 python3 tests/irc_event_fairness.py ./$(NAME)
 
 smoke: test
 
diff --git a/tests/irc_event_fairness.py b/tests/irc_event_fairness.py
new file mode 100644
index 0000000..147013e
--- /dev/null
+++ b/tests/irc_event_fairness.py
@@ -0,0 +1,242 @@
+#!/usr/bin/env python3
+"""Exercise high file-descriptor counts and non-blocking output isolation."""
+
+from __future__ import annotations
+
+import argparse
+import selectors
+import signal
+import socket
+import subprocess
+import tempfile
+import time
+from pathlib import Path
+from typing import Dict, List, Optional
+
+
+HOST = "127.0.0.1"
+PASSWORD = "event-fairness-secret"
+PEER_COUNT = 160
+FLOOD_FRAME_COUNT = 4096
+FLOOD_PAYLOAD = "x" * 400
+
+
+class Peer:
+    def __init__(self, port: int, label: str, receive_buffer: Optional[int] = None):
+        self.label = label
+        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
+        if receive_buffer is not None:
+            self.socket.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, receive_buffer)
+        self.socket.settimeout(10.0)
+        self.socket.connect((HOST, port))
+        self.buffer = b""
+
+    def send_line(self, line: str) -> None:
+        self.socket.sendall(line.encode("utf-8") + b"\r\n")
+
+    def send_raw(self, data: bytes) -> None:
+        self.socket.sendall(data)
+
+    def expect_line(self, expected: str, timeout: float = 10.0) -> None:
+        deadline = time.monotonic() + timeout
+        while time.monotonic() < deadline:
+            while b"\n" in self.buffer:
+                raw, self.buffer = self.buffer.split(b"\n", 1)
+                if raw.endswith(b"\r"):
+                    raw = raw[:-1]
+                actual = raw.decode("utf-8", "replace")
+                if actual == expected:
+                    return
+            self.socket.settimeout(max(0.05, deadline - time.monotonic()))
+            chunk = self.socket.recv(65536)
+            if not chunk:
+                raise AssertionError(
+                    f"{self.label}: connection closed before {expected!r}"
+                )
+            self.buffer += chunk
+        raise AssertionError(f"{self.label}: did not receive {expected!r}")
+
+    def close(self) -> None:
+        try:
+            self.socket.close()
+        except OSError:
+            pass
+
+
+def reserve_port() -> int:
+    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as candidate:
+        candidate.bind((HOST, 0))
+        return int(candidate.getsockname()[1])
+
+
+def wait_until_listening(process: subprocess.Popen[str], port: int) -> None:
+    deadline = time.monotonic() + 10.0
+    while time.monotonic() < deadline:
+        if process.poll() is not None:
+            raise AssertionError(f"server exited during startup with {process.returncode}")
+        try:
+            with socket.create_connection((HOST, port), timeout=0.1):
+                return
+        except OSError:
+            time.sleep(0.02)
+    raise AssertionError("server did not start listening")
+
+
+def check_many_connections(port: int) -> None:
+    peers: List[Peer] = []
+    selector = selectors.DefaultSelector()
+    buffers: Dict[socket.socket, bytes] = {}
+    expected: Dict[socket.socket, bytes] = {}
+    try:
+        for index in range(PEER_COUNT):
+            peer = Peer(port, f"peer-{index}")
+            peer.socket.setblocking(False)
+            peers.append(peer)
+            selector.register(peer.socket, selectors.EVENT_READ)
+            buffers[peer.socket] = b""
+            expected[peer.socket] = (
+                f":irc.relay.local PONG irc.relay.local fanout-{index}".encode("ascii")
+            )
+
+        for index, peer in enumerate(peers):
+            payload = f"PING :fanout-{index}\r\n".encode("ascii")
+            peer.socket.setblocking(True)
+            peer.socket.settimeout(10.0)
+            peer.socket.sendall(payload)
+            peer.socket.setblocking(False)
+
+        pending = set(expected)
+        deadline = time.monotonic() + 15.0
+        while pending and time.monotonic() < deadline:
+            for key, _ in selector.select(timeout=0.25):
+                sock = key.fileobj
+                if sock not in pending:
+                    continue
+                chunk = sock.recv(4096)
+                if not chunk:
+                    raise AssertionError("a fan-out peer closed before receiving PONG")
+                buffers[sock] += chunk
+                lines = buffers[sock].split(b"\n")
+                buffers[sock] = lines.pop()
+                if any(line.rstrip(b"\r") == expected[sock] for line in lines):
+                    pending.remove(sock)
+
+        if pending:
+            labels = [
+                peers[index].label
+                for index, peer in enumerate(peers)
+                if peer.socket in pending
+            ]
+            raise AssertionError(
+                f"{len(pending)} of {PEER_COUNT} peers did not receive PONG: "
+                + ", ".join(labels[:10])
+            )
+    finally:
+        selector.close()
+        for peer in peers:
+            peer.close()
+
+
+def register(peer: Peer, nick: str) -> None:
+    peer.send_raw(
+        (
+            f"PASS {PASSWORD}\r\n"
+            f"NICK {nick}\r\n"
+            f"USER {nick} 0 * :{nick} fairness peer\r\n"
+        ).encode("ascii")
+    )
+    peer.expect_line(
+        f":irc.relay.local 003 {nick} :This server is running a C++17 event backend"
+    )
+
+
+def check_slow_receiver_isolation(port: int) -> None:
+    slow = Peer(port, "slow", receive_buffer=1024)
+    sender = Peer(port, "sender")
+    probe = Peer(port, "probe")
+    try:
+        register(slow, "slow")
+        register(sender, "sender")
+        register(probe, "probe")
+
+        flood = (
+            f"PRIVMSG slow :{FLOOD_PAYLOAD}\r\n".encode("ascii")
+            * FLOOD_FRAME_COUNT
+        )
+        sender.send_raw(flood)
+
+        token = "unrelated-connection-still-progresses"
+        probe.send_line(f"PING :{token}")
+        probe.expect_line(
+            f":irc.relay.local PONG irc.relay.local {token}",
+            timeout=15.0,
+        )
+    finally:
+        slow.close()
+        sender.close()
+        probe.close()
+
+
+def stop_server(process: subprocess.Popen[str]) -> None:
+    if process.poll() is None:
+        process.send_signal(signal.SIGTERM)
+    try:
+        process.wait(timeout=10.0)
+    except subprocess.TimeoutExpired:
+        process.kill()
+        process.wait(timeout=5.0)
+        raise AssertionError("server did not stop after SIGTERM")
+    if process.returncode != 0:
+        raise AssertionError(f"server exited with {process.returncode}")
+
+
+def run(binary: Path) -> None:
+    port = reserve_port()
+    with tempfile.TemporaryFile(mode="w+", encoding="utf-8") as log:
+        process = subprocess.Popen(
+            [
+                str(binary),
+                str(port),
+                PASSWORD,
+                "--idle-timeout=60",
+                "--ping-timeout=10",
+                "--registration-timeout=30",
+                "--rate-limit=10000:60",
+                "--max-pending-bytes=16777216",
+                "--max-connections=256",
+            ],
+            stdout=log,
+            stderr=subprocess.STDOUT,
+            text=True,
+        )
+        try:
+            wait_until_listening(process, port)
+            check_many_connections(port)
+            check_slow_receiver_isolation(port)
+            stop_server(process)
+        except BaseException:
+            if process.poll() is None:
+                process.kill()
+                process.wait(timeout=5.0)
+            log.seek(0)
+            output = log.read()
+            if output:
+                print("server output:", flush=True)
+                print(output, flush=True)
+            raise
+
+
+def main() -> int:
+    parser = argparse.ArgumentParser()
+    parser.add_argument("binary", type=Path)
+    args = parser.parse_args()
+    run(args.binary.resolve())
+    print(
+        f"event fairness passed with {PEER_COUNT} simultaneous connections "
+        "and one unread recipient"
+    )
+    return 0
+
+
+if __name__ == "__main__":
+    raise SystemExit(main())


## `ci: Linux·macOS 회귀와 새니타이저 자동화`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..81776cb
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,51 @@
+name: IRC 회귀 검사
+
+on:
+  push:
+  pull_request:
+
+permissions:
+  contents: read
+
+jobs:
+  platform-regression:
+    name: ${{ matrix.os }} 빌드와 회귀
+    runs-on: ${{ matrix.os }}
+    timeout-minutes: 20
+    strategy:
+      fail-fast: false
+      matrix:
+        os:
+          - ubuntu-latest
+          - macos-latest
+
+    steps:
+      - name: 저장소 가져오기
+        uses: actions/checkout@v4
+
+      - name: 경고를 오류로 처리하여 빌드
+        run: make -j2
+
+      - name: 단위 및 네트워크 회귀 실행
+        run: make test
+
+  linux-sanitizers:
+    name: Linux ASan·UBSan
+    runs-on: ubuntu-latest
+    timeout-minutes: 20
+    env:
+      SANITIZER_FLAGS: >-
+        -std=c++17 -Wall -Wextra -Werror -g -O1
+        -fno-omit-frame-pointer -fsanitize=address,undefined
+      ASAN_OPTIONS: detect_leaks=1:halt_on_error=1:abort_on_error=1
+      UBSAN_OPTIONS: print_stacktrace=1:halt_on_error=1
+
+    steps:
+      - name: 저장소 가져오기
+        uses: actions/checkout@v4
+
+      - name: ASan·UBSan 빌드
+        run: make -j2 CXXFLAGS="${SANITIZER_FLAGS}"
+
+      - name: 새니타이저 단위 및 네트워크 회귀 실행
+        run: make test CXXFLAGS="${SANITIZER_FLAGS}"


## `build: expose deterministic verification targets`

diff --git a/Makefile b/Makefile
index bfe6f3b..f5e4971 100644
--- a/Makefile
+++ b/Makefile
@@ -4,8 +4,14 @@ SERVER_LIFETIME_TEST := tests/server_lifetime_test
 APPLICATION_LIFETIME_TEST := tests/application_lifetime_test
 
 CXX ?= c++
-CXXFLAGS ?= -std=c++17 -Wall -Wextra -Werror -g
+BASE_CXXFLAGS := -std=c++17 -Wall -Wextra -Werror -g
+EXTRA_CXXFLAGS ?=
+CXXFLAGS ?= $(BASE_CXXFLAGS) $(EXTRA_CXXFLAGS)
 CPPFLAGS := -Iinclude
+BASH ?= bash
+PYTHON ?= python3
+SANITIZER_FLAGS ?= -O1 -fno-omit-frame-pointer \
+	-fsanitize=address,undefined
 
 UNAME_S := $(shell uname -s)
 
@@ -25,8 +31,10 @@ SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp src/Mes
 	src/Connection.cpp src/Server.cpp $(EVENT_SRC)
 OBJS := $(SRCS:.cpp=.o)
 DEPS := $(OBJS:.o=.d)
+PROJECT_HEADERS := $(sort $(wildcard include/*.hpp src/*.hpp))
 
-.PHONY: all application-test clean connection-test event-test fclean re smoke test unit
+.PHONY: all application-test check ci clean connection-test event-test \
+	fclean help re sanitize smoke test unit
 
 all: $(NAME)
 
@@ -36,13 +44,15 @@ $(NAME): $(OBJS)
 %.o: %.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -MMD -MP -c $< -o $@
 
-$(CONNECTION_TEST): tests/connection_test.cpp src/Connection.cpp include/Connection.hpp src/ConnectionLimits.hpp
+$(CONNECTION_TEST): tests/connection_test.cpp src/Connection.cpp \
+		$(PROJECT_HEADERS)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) tests/connection_test.cpp src/Connection.cpp -o $@
 
 connection-test: $(CONNECTION_TEST)
 	./$(CONNECTION_TEST)
 
-$(SERVER_LIFETIME_TEST): tests/server_lifetime_test.cpp src/Connection.cpp src/Server.cpp $(EVENT_SRC) include/Server.hpp include/Connection.hpp include/EventManager.hpp src/ConnectionLimits.hpp
+$(SERVER_LIFETIME_TEST): tests/server_lifetime_test.cpp src/Connection.cpp \
+		src/Server.cpp $(EVENT_SRC) $(PROJECT_HEADERS)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) tests/server_lifetime_test.cpp src/Connection.cpp src/Server.cpp $(EVENT_SRC) -o $@
 
 unit: $(SERVER_LIFETIME_TEST)
@@ -50,21 +60,45 @@ unit: $(SERVER_LIFETIME_TEST)
 
 APP_TEST_SRCS := $(filter-out src/main.cpp,$(SRCS))
 
-$(APPLICATION_LIFETIME_TEST): tests/application_lifetime_test.cpp $(APP_TEST_SRCS) src/IrcApplication.hpp src/ClientRegistry.hpp
+$(APPLICATION_LIFETIME_TEST): tests/application_lifetime_test.cpp \
+		$(APP_TEST_SRCS) $(PROJECT_HEADERS)
 	$(CXX) $(CPPFLAGS) -Isrc $(CXXFLAGS) tests/application_lifetime_test.cpp $(APP_TEST_SRCS) -o $@
 
 application-test: $(APPLICATION_LIFETIME_TEST)
 	./$(APPLICATION_LIFETIME_TEST)
 
 test: all connection-test unit application-test
-	bash tests/irc_smoke.sh
+	PYTHON="$(PYTHON)" $(BASH) tests/irc_smoke.sh
 	$(MAKE) event-test
 
 event-test: all
-	PYTHONDONTWRITEBYTECODE=1 python3 tests/irc_event_fairness.py ./$(NAME)
+	PYTHONDONTWRITEBYTECODE=1 $(PYTHON) tests/irc_event_fairness.py ./$(NAME)
 
 smoke: test
 
+check: test
+
+ci:
+	$(MAKE) fclean
+	$(MAKE) check
+
+sanitize:
+	$(MAKE) fclean
+	$(MAKE) EXTRA_CXXFLAGS="$(SANITIZER_FLAGS)" check
+
+help:
+	@printf '%s\n' \
+		'Targets:' \
+		'  all       Build the IRC relay server.' \
+		'  test      Run unit, protocol, and event-fairness tests.' \
+		'  check     Run the complete functional test gate.' \
+		'  ci        Clean, rebuild, and run the functional gate.' \
+		'  sanitize  Clean and run the gate with ASan and UBSan.' \
+		'  clean     Remove objects, dependencies, and test binaries.' \
+		'  fclean    Remove all generated build outputs.' \
+		'' \
+		'Overrides: CXX, CXXFLAGS, EXTRA_CXXFLAGS, CPPFLAGS, BASH, PYTHON'
+
 clean:
 	rm -f $(OBJS) $(DEPS) $(CONNECTION_TEST) $(SERVER_LIFETIME_TEST) $(APPLICATION_LIFETIME_TEST)
 	rm -rf $(CONNECTION_TEST).dSYM $(SERVER_LIFETIME_TEST).dSYM $(APPLICATION_LIFETIME_TEST).dSYM
diff --git a/tests/irc_smoke.sh b/tests/irc_smoke.sh
index 7a25e5d..69d971d 100755
--- a/tests/irc_smoke.sh
+++ b/tests/irc_smoke.sh
@@ -3,7 +3,8 @@ set -euo pipefail
 export PYTHONDONTWRITEBYTECODE=1
 
 ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
-PORT="${IRC_SMOKE_PORT:-$(python3 - <<'PY'
+PYTHON_BIN="${PYTHON:-python3}"
+PORT="${IRC_SMOKE_PORT:-$("${PYTHON_BIN}" - <<'PY'
 import socket
 s = socket.socket()
 s.bind(("127.0.0.1", 0))
@@ -16,11 +17,22 @@ LOG_FILE="$(mktemp "${TMPDIR:-/tmp}/irc_smoke_server.XXXXXX")"
 SERVER_PID=""
 
 cleanup() {
+	local status=$?
+	set +e
 	if [[ -n "${SERVER_PID}" ]] && kill -0 "${SERVER_PID}" 2>/dev/null; then
 		kill "${SERVER_PID}" 2>/dev/null || true
 		wait "${SERVER_PID}" 2>/dev/null || true
 	fi
+	if [[ ${status} -ne 0 ]]; then
+		printf '%s\n' 'IRC smoke server log:' >&2
+		if [[ -s "${LOG_FILE}" ]]; then
+			cat "${LOG_FILE}" >&2
+		else
+			printf '%s\n' '(server log was empty)' >&2
+		fi
+	fi
 	rm -f "${LOG_FILE}"
+	return "${status}"
 }
 trap cleanup EXIT
 
@@ -34,7 +46,7 @@ make -C "${ROOT}" >/dev/null
 	>"${LOG_FILE}" 2>&1 &
 SERVER_PID="$!"
 
-python3 - <<PY
+"${PYTHON_BIN}" - <<PY
 import socket
 import time
 
@@ -50,8 +62,8 @@ while time.time() < deadline:
 raise SystemExit("server did not accept connections")
 PY
 
-python3 "${ROOT}/tools/irc_smoke_client.py" 127.0.0.1 "${PORT}" "${PASSWORD}"
-python3 "${ROOT}/tests/irc_contract.py" \
+"${PYTHON_BIN}" "${ROOT}/tools/irc_smoke_client.py" 127.0.0.1 "${PORT}" "${PASSWORD}"
+"${PYTHON_BIN}" "${ROOT}/tests/irc_contract.py" \
 	"${ROOT}/irc-relay-server" 127.0.0.1 "${PORT}" "${PASSWORD}" \
 	"${SERVER_PID}" "${LOG_FILE}"
 wait "${SERVER_PID}"


