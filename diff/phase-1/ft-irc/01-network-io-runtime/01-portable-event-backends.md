# 이식 가능한 이벤트 백엔드

## `feat(event): 이벤트 준비 상태 계약 정의`

diff --git a/include/EventManager.hpp b/include/EventManager.hpp
new file mode 100644
index 0000000..82822d3
--- /dev/null
+++ b/include/EventManager.hpp
@@ -0,0 +1,64 @@
+#ifndef IRC_EVENT_MANAGER_HPP
+#define IRC_EVENT_MANAGER_HPP
+
+#include <memory>
+#include <vector>
+
+namespace irc {
+
+enum class EventInterest : unsigned int {
+    None = 0,
+    Read = 1u << 0,
+    Write = 1u << 1
+};
+
+constexpr EventInterest operator|(EventInterest lhs, EventInterest rhs) noexcept
+{
+    return static_cast<EventInterest>(
+        static_cast<unsigned int>(lhs) | static_cast<unsigned int>(rhs));
+}
+
+constexpr EventInterest operator&(EventInterest lhs, EventInterest rhs) noexcept
+{
+    return static_cast<EventInterest>(
+        static_cast<unsigned int>(lhs) & static_cast<unsigned int>(rhs));
+}
+
+inline EventInterest& operator|=(EventInterest& lhs, EventInterest rhs) noexcept
+{
+    lhs = lhs | rhs;
+    return lhs;
+}
+
+constexpr bool hasInterest(EventInterest interests, EventInterest expected) noexcept
+{
+    return static_cast<unsigned int>(interests & expected) != 0;
+}
+
+struct Event {
+    int fd = -1;
+    EventInterest interests = EventInterest::None;
+    bool error = false;
+    bool hangup = false;
+    int errorCode = 0;
+};
+
+class EventManager {
+public:
+    virtual ~EventManager() = default;
+
+    static std::unique_ptr<EventManager> createDefault();
+
+    virtual void addFd(int fd, EventInterest interests) = 0;
+    virtual void updateFd(int fd, EventInterest interests) = 0;
+    virtual void removeFd(int fd) = 0;
+    virtual std::vector<Event> wait(int timeoutMs) = 0;
+};
+
+} // namespace irc
+
+using Event = irc::Event;
+using EventInterest = irc::EventInterest;
+using EventManager = irc::EventManager;
+
+#endif // IRC_EVENT_MANAGER_HPP


## `feat(event): kqueue 관심 상태 등록 구현`

diff --git a/src/KqueueEventManager.cpp b/src/KqueueEventManager.cpp
new file mode 100644
index 0000000..06003c1
--- /dev/null
+++ b/src/KqueueEventManager.cpp
@@ -0,0 +1,123 @@
+#if defined(__APPLE__) || defined(__FreeBSD__) || defined(__NetBSD__) || defined(__OpenBSD__)
+
+#include "EventManager.hpp"
+
+#include <sys/event.h>
+#include <unistd.h>
+
+#include <cerrno>
+#include <system_error>
+#include <unordered_map>
+
+namespace irc {
+namespace {
+
+class KqueueEventManager : public EventManager {
+public:
+    KqueueEventManager()
+        : kqueueFd_(::kqueue())
+    {
+        if (kqueueFd_ == -1) {
+            throw std::system_error(errno, std::generic_category(), "kqueue");
+        }
+    }
+
+    ~KqueueEventManager()
+    {
+        if (kqueueFd_ != -1) {
+            ::close(kqueueFd_);
+        }
+    }
+
+    void addFd(int fd, EventInterest interests) override
+    {
+        if (interests == EventInterest::None) {
+            return;
+        }
+        if (interests_.find(fd) != interests_.end()) {
+            updateFd(fd, interests);
+            return;
+        }
+        applyInterestChange(fd, EventInterest::None, interests);
+        interests_[fd] = interests;
+    }
+
+    void updateFd(int fd, EventInterest interests) override
+    {
+        const std::unordered_map<int, EventInterest>::iterator found = interests_.find(fd);
+        const EventInterest oldInterests =
+            found == interests_.end() ? EventInterest::None : found->second;
+
+        if (interests == EventInterest::None) {
+            removeFd(fd);
+            return;
+        }
+
+        applyInterestChange(fd, oldInterests, interests);
+        interests_[fd] = interests;
+    }
+
+    void removeFd(int fd) override
+    {
+        const std::unordered_map<int, EventInterest>::iterator found = interests_.find(fd);
+        if (found == interests_.end()) {
+            return;
+        }
+
+        removeFilterIfWatched(fd, found->second, EventInterest::Read, EVFILT_READ);
+        removeFilterIfWatched(fd, found->second, EventInterest::Write, EVFILT_WRITE);
+        interests_.erase(found);
+    }
+
+private:
+    int kqueueFd_;
+    std::unordered_map<int, EventInterest> interests_;
+
+    void applyInterestChange(int fd, EventInterest oldInterests, EventInterest newInterests)
+    {
+        updateFilterIfChanged(fd, oldInterests, newInterests, EventInterest::Read, EVFILT_READ);
+        updateFilterIfChanged(fd, oldInterests, newInterests, EventInterest::Write, EVFILT_WRITE);
+    }
+
+    void updateFilterIfChanged(
+        int fd,
+        EventInterest oldInterests,
+        EventInterest newInterests,
+        EventInterest interest,
+        int16_t filter)
+    {
+        const bool hadInterest = hasInterest(oldInterests, interest);
+        const bool wantsInterest = hasInterest(newInterests, interest);
+        if (hadInterest == wantsInterest) {
+            return;
+        }
+
+        const uint16_t flags = wantsInterest ? (EV_ADD | EV_ENABLE) : EV_DELETE;
+        applyFilterChange(fd, filter, flags, false);
+    }
+
+    void removeFilterIfWatched(int fd, EventInterest interests, EventInterest interest, int16_t filter)
+    {
+        if (hasInterest(interests, interest)) {
+            applyFilterChange(fd, filter, EV_DELETE, true);
+        }
+    }
+
+    void applyFilterChange(int fd, int16_t filter, uint16_t flags, bool ignoreMissing)
+    {
+        struct kevent change;
+        EV_SET(&change, static_cast<uintptr_t>(fd), filter, flags, 0, 0, NULL);
+
+        if (::kevent(kqueueFd_, &change, 1, NULL, 0, NULL) == -1) {
+            if (ignoreMissing && (errno == ENOENT || errno == EBADF)) {
+                return;
+            }
+            throw std::system_error(errno, std::generic_category(), "kevent update");
+        }
+    }
+};
+
+} // namespace
+} // namespace irc
+
+#endif


## `feat(event): kqueue 준비 이벤트 변환 구현`

diff --git a/src/KqueueEventManager.cpp b/src/KqueueEventManager.cpp
index 06003c1..bc192fe 100644
--- a/src/KqueueEventManager.cpp
+++ b/src/KqueueEventManager.cpp
@@ -3,9 +3,13 @@
 #include "EventManager.hpp"
 
 #include <sys/event.h>
+#include <sys/time.h>
 #include <unistd.h>
 
+#include <array>
 #include <cerrno>
+#include <cstring>
+#include <stdexcept>
 #include <system_error>
 #include <unordered_map>
 
@@ -69,6 +73,51 @@ public:
         interests_.erase(found);
     }
 
+    std::vector<Event> wait(int timeoutMs) override
+    {
+        std::array<struct kevent, 128> nativeEvents;
+        struct timespec timeout;
+        struct timespec* timeoutPtr = NULL;
+
+        if (timeoutMs >= 0) {
+            timeout.tv_sec = timeoutMs / 1000;
+            timeout.tv_nsec = (timeoutMs % 1000) * 1000000;
+            timeoutPtr = &timeout;
+        }
+
+        const int count =
+            ::kevent(kqueueFd_, NULL, 0, nativeEvents.data(), nativeEvents.size(), timeoutPtr);
+        if (count == -1) {
+            if (errno == EINTR) {
+                return std::vector<Event>();
+            }
+            throw std::system_error(errno, std::generic_category(), "kevent wait");
+        }
+
+        std::vector<Event> events;
+        events.reserve(static_cast<std::size_t>(count));
+        for (int i = 0; i < count; ++i) {
+            Event event;
+            event.fd = static_cast<int>(nativeEvents[static_cast<std::size_t>(i)].ident);
+            event.error = (nativeEvents[static_cast<std::size_t>(i)].flags & EV_ERROR) != 0;
+            event.hangup = (nativeEvents[static_cast<std::size_t>(i)].flags & EV_EOF) != 0;
+            if (event.error) {
+                event.errorCode = static_cast<int>(nativeEvents[static_cast<std::size_t>(i)].data);
+            } else if (event.hangup) {
+                event.errorCode = static_cast<int>(nativeEvents[static_cast<std::size_t>(i)].fflags);
+            }
+
+            if (nativeEvents[static_cast<std::size_t>(i)].filter == EVFILT_READ) {
+                event.interests |= EventInterest::Read;
+            } else if (nativeEvents[static_cast<std::size_t>(i)].filter == EVFILT_WRITE) {
+                event.interests |= EventInterest::Write;
+            }
+
+            events.push_back(event);
+        }
+        return events;
+    }
+
 private:
     int kqueueFd_;
     std::unordered_map<int, EventInterest> interests_;
@@ -118,6 +167,12 @@ private:
 };
 
 } // namespace
+
+std::unique_ptr<EventManager> EventManager::createDefault()
+{
+    return std::unique_ptr<EventManager>(new KqueueEventManager());
+}
+
 } // namespace irc
 
 #endif


## `feat(event): epoll 관심 상태 등록 구현`

diff --git a/src/EpollEventManager.cpp b/src/EpollEventManager.cpp
new file mode 100644
index 0000000..f4b1067
--- /dev/null
+++ b/src/EpollEventManager.cpp
@@ -0,0 +1,116 @@
+#if defined(__linux__)
+
+#include "EventManager.hpp"
+
+#include <sys/epoll.h>
+#include <unistd.h>
+
+#include <cerrno>
+#include <cstdint>
+#include <system_error>
+#include <unordered_map>
+
+namespace irc {
+namespace {
+
+int nativeEventsFor(EventInterest interests)
+{
+    int events = EPOLLERR | EPOLLHUP;
+    if (hasInterest(interests, EventInterest::Read)) {
+        events |= EPOLLIN;
+#ifdef EPOLLRDHUP
+        events |= EPOLLRDHUP;
+#endif
+    }
+    if (hasInterest(interests, EventInterest::Write)) {
+        events |= EPOLLOUT;
+    }
+    return events;
+}
+
+class EpollEventManager : public EventManager {
+public:
+    EpollEventManager()
+        : epollFd_(::epoll_create1(EPOLL_CLOEXEC))
+    {
+        if (epollFd_ == -1) {
+            throw std::system_error(errno, std::generic_category(), "epoll_create1");
+        }
+    }
+
+    ~EpollEventManager()
+    {
+        if (epollFd_ != -1) {
+            ::close(epollFd_);
+        }
+    }
+
+    void addFd(int fd, EventInterest interests) override
+    {
+        if (interests == EventInterest::None) {
+            return;
+        }
+        if (interests_.find(fd) != interests_.end()) {
+            updateFd(fd, interests);
+            return;
+        }
+
+        control(fd, EPOLL_CTL_ADD, interests);
+        interests_[fd] = interests;
+    }
+
+    void updateFd(int fd, EventInterest interests) override
+    {
+        if (interests == EventInterest::None) {
+            removeFd(fd);
+            return;
+        }
+
+        const std::unordered_map<int, EventInterest>::iterator found = interests_.find(fd);
+        if (found == interests_.end()) {
+            addFd(fd, interests);
+            return;
+        }
+
+        control(fd, EPOLL_CTL_MOD, interests);
+        found->second = interests;
+    }
+
+    void removeFd(int fd) override
+    {
+        const std::unordered_map<int, EventInterest>::iterator found = interests_.find(fd);
+        if (found == interests_.end()) {
+            return;
+        }
+
+        struct epoll_event event;
+        event.events = 0;
+        event.data.fd = fd;
+        if (::epoll_ctl(epollFd_, EPOLL_CTL_DEL, fd, &event) == -1) {
+            if (errno != ENOENT && errno != EBADF) {
+                throw std::system_error(errno, std::generic_category(), "epoll_ctl del");
+            }
+        }
+        interests_.erase(found);
+    }
+
+private:
+    int epollFd_;
+    std::unordered_map<int, EventInterest> interests_;
+
+    void control(int fd, int operation, EventInterest interests)
+    {
+        struct epoll_event event;
+        event.events = static_cast<uint32_t>(nativeEventsFor(interests));
+        event.data.fd = fd;
+
+        if (::epoll_ctl(epollFd_, operation, fd, &event) == -1) {
+            throw std::system_error(errno, std::generic_category(), "epoll_ctl");
+        }
+    }
+};
+
+} // namespace
+} // namespace irc
+
+#endif


## `feat(event): epoll 준비 이벤트 변환 구현`

diff --git a/src/EpollEventManager.cpp b/src/EpollEventManager.cpp
index f4b1067..a2ee4b5 100644
--- a/src/EpollEventManager.cpp
+++ b/src/EpollEventManager.cpp
@@ -3,8 +3,10 @@
 #include "EventManager.hpp"
 
 #include <sys/epoll.h>
+#include <sys/socket.h>
 #include <unistd.h>
 
+#include <array>
 #include <cerrno>
 #include <cstdint>
 #include <system_error>
@@ -28,6 +30,16 @@ int nativeEventsFor(EventInterest interests)
     return events;
 }
 
+int socketErrorFor(int fd)
+{
+    int error = 0;
+    socklen_t length = sizeof(error);
+    if (::getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &length) == -1) {
+        return errno;
+    }
+    return error;
+}
+
 class EpollEventManager : public EventManager {
 public:
     EpollEventManager()
@@ -94,6 +106,46 @@ public:
         interests_.erase(found);
     }
 
+    std::vector<Event> wait(int timeoutMs) override
+    {
+        std::array<struct epoll_event, 128> nativeEvents;
+        const int count =
+            ::epoll_wait(epollFd_, nativeEvents.data(), nativeEvents.size(), timeoutMs);
+
+        if (count == -1) {
+            if (errno == EINTR) {
+                return std::vector<Event>();
+            }
+            throw std::system_error(errno, std::generic_category(), "epoll_wait");
+        }
+
+        std::vector<Event> events;
+        events.reserve(static_cast<std::size_t>(count));
+        for (int i = 0; i < count; ++i) {
+            const uint32_t native = nativeEvents[static_cast<std::size_t>(i)].events;
+
+            Event event;
+            event.fd = nativeEvents[static_cast<std::size_t>(i)].data.fd;
+            event.error = (native & EPOLLERR) != 0;
+            event.hangup = (native & EPOLLHUP) != 0;
+#ifdef EPOLLRDHUP
+            event.hangup = event.hangup || ((native & EPOLLRDHUP) != 0);
+#endif
+            if (event.error) {
+                event.errorCode = socketErrorFor(event.fd);
+            }
+            if ((native & EPOLLIN) != 0) {
+                event.interests |= EventInterest::Read;
+            }
+            if ((native & EPOLLOUT) != 0) {
+                event.interests |= EventInterest::Write;
+            }
+
+            events.push_back(event);
+        }
+        return events;
+    }
+
 private:
     int epollFd_;
     std::unordered_map<int, EventInterest> interests_;
@@ -111,6 +163,12 @@ private:
 };
 
 } // namespace
+
+std::unique_ptr<EventManager> EventManager::createDefault()
+{
+    return std::unique_ptr<EventManager>(new EpollEventManager());
+}
+
 } // namespace irc
 
 #endif


## `build(project): 플랫폼별 IRC 서버 빌드 구성`

diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..9263916
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,44 @@
+NAME := irc-relay-server
+
+CXX ?= c++
+CXXFLAGS ?= -std=c++17 -Wall -Wextra -Werror -g
+CPPFLAGS := -Iinclude
+
+UNAME_S := $(shell uname -s)
+
+ifeq ($(UNAME_S),Darwin)
+EVENT_SRC := src/KqueueEventManager.cpp
+CPPFLAGS += -DIRC_USE_KQUEUE
+else ifeq ($(UNAME_S),Linux)
+EVENT_SRC := src/EpollEventManager.cpp
+CPPFLAGS += -DIRC_USE_EPOLL
+else
+$(error Unsupported OS for IRC event backend: $(UNAME_S))
+endif
+
+SRCS := src/main.cpp src/IrcApplication.cpp src/RegistrationCommands.cpp \
+	src/ApplicationSupport.cpp src/ClientRegistry.cpp src/RuntimeConfig.cpp \
+	src/IrcMessage.cpp src/Channel.cpp src/Replies.cpp \
+	src/Connection.cpp src/Server.cpp $(EVENT_SRC)
+OBJS := $(SRCS:.cpp=.o)
+DEPS := $(OBJS:.o=.d)
+
+.PHONY: all clean fclean re
+
+all: $(NAME)
+
+$(NAME): $(OBJS)
+	$(CXX) $(CXXFLAGS) $(OBJS) -o $@
+
+%.o: %.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -MMD -MP -c $< -o $@
+
+clean:
+	rm -f $(OBJS) $(DEPS)
+
+fclean: clean
+	rm -f $(NAME)
+
+re: fclean all
+
+-include $(DEPS)
