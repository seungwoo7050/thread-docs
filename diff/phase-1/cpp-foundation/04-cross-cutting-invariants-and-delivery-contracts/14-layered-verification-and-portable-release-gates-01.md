# 계층형 검증과 이식 가능한 릴리스 게이트

## `build(makefile): C++98 정적 라이브러리 빌드 구성`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..4852624
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,5 @@
+/build/
+/bin/
+*.a
+*.dSYM/
+.DS_Store
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..e322486
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,43 @@
+NAME := libcpp_foundation.a
+
+CXX := c++
+override CXXFLAGS := -Wall -Wextra -Werror -Wpedantic -pedantic-errors \
+	-std=c++98 -Wold-style-cast -Wcast-qual -Woverloaded-virtual \
+	-Wnon-virtual-dtor -Wc++11-extensions
+override CPPFLAGS := -Iinclude
+DEPFLAGS := -MMD -MP
+AR := ar
+ARFLAGS := rcs
+RM := rm -f
+RMDIR := rm -rf
+MKDIR := mkdir -p
+
+SRC := $(sort $(wildcard src/*.cpp))
+OBJ := $(SRC:src/%.cpp=build/obj/%.o)
+DEP := $(OBJ:.o=.d)
+
+.PHONY: all clean fclean re
+
+all: $(NAME)
+
+$(NAME): $(OBJ)
+	$(RM) $@
+	@if test -n "$(strip $(OBJ))"; then \
+		$(AR) $(ARFLAGS) $@ $(OBJ); \
+	else \
+		printf '!<arch>\n' > $@; \
+	fi
+
+build/obj/%.o: src/%.cpp
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(DEPFLAGS) -c $< -o $@
+
+clean:
+	$(RMDIR) build
+
+fclean: clean
+	$(RM) $(NAME)
+
+re: fclean all
+
+-include $(DEP)


## `test(contact): 공개 계약과 명령행 세션 검증`

diff --git a/Makefile b/Makefile
index 8d03209..36fae6a 100644
--- a/Makefile
+++ b/Makefile
@@ -22,7 +22,7 @@ APP_BIN := $(APP_SRC:apps/%.cpp=bin/%)
 TEST_SRC := $(sort $(wildcard tests/test_*.cpp))
 TEST_BIN := build/tests/unit
 
-.PHONY: all test check clean fclean re
+.PHONY: all test-unit test-contract test-integration test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -42,9 +42,20 @@ $(TEST_BIN): $(TEST_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(TEST_SRC) $(NAME) -o $@
 
-test: $(TEST_BIN)
+test-unit: $(TEST_BIN)
 	./$(TEST_BIN)
 
+test-contract:
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/contact_headers.cpp
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
+
+test-integration: bin/ex00_contact_book
+	sh tests/check_cli.sh
+
+test: test-unit test-contract test-integration
+
 check:
 	git diff --check
 	$(MAKE) fclean
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
new file mode 100644
index 0000000..7fcc8c8
--- /dev/null
+++ b/tests/check_cli.sh
@@ -0,0 +1,10 @@
+#!/bin/sh
+
+set -eu
+
+temporary_directory=$(mktemp -d "${TMPDIR:-/tmp}/cpp-foundation-cli.XXXXXX")
+trap 'rm -rf "$temporary_directory"' EXIT HUP INT TERM
+
+./bin/ex00_contact_book < tests/fixtures/contact-session.in \
+    > "$temporary_directory/contact.out"
+diff -u tests/fixtures/contact-session.out "$temporary_directory/contact.out"
diff --git a/tests/compile/contact_headers.cpp b/tests/compile/contact_headers.cpp
new file mode 100644
index 0000000..50bee40
--- /dev/null
+++ b/tests/compile/contact_headers.cpp
@@ -0,0 +1,17 @@
+#include "cppf/Contact.hpp"
+#include "cppf/Contact.hpp"
+#include "cppf/ContactBook.hpp"
+#include "cppf/ContactBook.hpp"
+
+#include <sstream>
+
+int main()
+{
+    cppf::ContactBook book;
+    const cppf::Contact contact("Ada", "math");
+    std::ostringstream output;
+
+    book.add(contact);
+    book.write(output);
+    return book.at(0).name() == "Ada" ? 0 : 1;
+}
diff --git a/tests/compile/contact_private_fail.cpp b/tests/compile/contact_private_fail.cpp
new file mode 100644
index 0000000..2ea8f11
--- /dev/null
+++ b/tests/compile/contact_private_fail.cpp
@@ -0,0 +1,8 @@
+#include "cppf/Contact.hpp"
+
+int main()
+{
+    cppf::Contact contact;
+
+    return contact.name_.empty() ? 0 : 1;
+}
diff --git a/tests/fixtures/contact-session.in b/tests/fixtures/contact-session.in
new file mode 100644
index 0000000..c106538
--- /dev/null
+++ b/tests/fixtures/contact-session.in
@@ -0,0 +1,5 @@
+ADD Ada|math
+ADD Grace|compiler
+BAD
+LIST
+QUIT
diff --git a/tests/fixtures/contact-session.out b/tests/fixtures/contact-session.out
new file mode 100644
index 0000000..de6c0f4
--- /dev/null
+++ b/tests/fixtures/contact-session.out
@@ -0,0 +1,5 @@
+ok
+ok
+error
+0|Ada|math
+1|Grace|compiler
diff --git a/tests/test_contact_book.cpp b/tests/test_contact_book.cpp
index 02d80ec..192632d 100644
--- a/tests/test_contact_book.cpp
+++ b/tests/test_contact_book.cpp
@@ -1,6 +1,7 @@
 #include "cppf/ContactBook.hpp"
 #include "support/Test.hpp"
 
+#include <sstream>
 #include <stdexcept>
 
 void testContactBook(test_support::Suite &suite)
@@ -11,6 +12,7 @@ void testContactBook(test_support::Suite &suite)
     };
     std::size_t index;
     bool threw;
+    std::ostringstream output;
 
     suite.check(book.size() == 0, "contact book starts empty");
     book.add(cppf::Contact());
@@ -22,6 +24,13 @@ void testContactBook(test_support::Suite &suite)
     suite.check(book.at(0).name() == "C", "contact book replaces oldest");
     suite.check(book.at(7).name() == "J", "contact book keeps newest");
     suite.check(book.at(3).name() == "F", "contact book maps logical order");
+    suite.check(book.at(0).name() != "A", "contact book discards first value");
+    suite.check(book.at(0).name() != "B", "contact book discards second value");
+    book.write(output);
+    suite.check(output.str() ==
+                    "0|C|note\n1|D|note\n2|E|note\n3|F|note\n"
+                    "4|G|note\n5|H|note\n6|I|note\n7|J|note\n",
+                "contact book writes logical order");
 
     threw = false;
     try


## `test(buffer): 할당 실패와 복사 생략 비활성화 검증`

diff --git a/Makefile b/Makefile
index 36fae6a..5010050 100644
--- a/Makefile
+++ b/Makefile
@@ -21,8 +21,13 @@ APP_BIN := $(APP_SRC:apps/%.cpp=bin/%)
 
 TEST_SRC := $(sort $(wildcard tests/test_*.cpp))
 TEST_BIN := build/tests/unit
+FAILURE_BIN := build/tests/buffer_failure
+FAILURE_SRC := tests/failure/test_buffer_failure.cpp \
+	tests/support/FailingNew.cpp
+NO_ELIDE_BIN := build/tests/unit_no_elide
 
-.PHONY: all test-unit test-contract test-integration test check clean fclean re
+.PHONY: all test-unit failure-test test-no-elide test-contract \
+	test-integration test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -45,6 +50,21 @@ $(TEST_BIN): $(TEST_SRC) $(NAME)
 test-unit: $(TEST_BIN)
 	./$(TEST_BIN)
 
+$(FAILURE_BIN): $(FAILURE_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(FAILURE_SRC) $(NAME) -o $@
+
+failure-test: $(FAILURE_BIN)
+	./$(FAILURE_BIN)
+
+$(NO_ELIDE_BIN): $(TEST_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fno-elide-constructors \
+		$(TEST_SRC) $(NAME) -o $@
+
+test-no-elide: $(NO_ELIDE_BIN)
+	./$(NO_ELIDE_BIN)
+
 test-contract:
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_headers.cpp
@@ -54,7 +74,7 @@ test-contract:
 test-integration: bin/ex00_contact_book
 	sh tests/check_cli.sh
 
-test: test-unit test-contract test-integration
+test: test-unit failure-test test-no-elide test-contract test-integration
 
 check:
 	git diff --check
diff --git a/tests/failure/test_buffer_failure.cpp b/tests/failure/test_buffer_failure.cpp
new file mode 100644
index 0000000..883c31b
--- /dev/null
+++ b/tests/failure/test_buffer_failure.cpp
@@ -0,0 +1,159 @@
+#include "cppf/TextBuffer.hpp"
+#include "support/FailingNew.hpp"
+
+#include <cstring>
+#include <iostream>
+#include <new>
+
+namespace
+{
+
+unsigned int checks = 0;
+unsigned int failures = 0;
+
+void check(bool condition)
+{
+    ++checks;
+    if (!condition)
+        ++failures;
+}
+
+void testConstructionFailure()
+{
+    const std::size_t baseline = failing_new::liveBlocks();
+    bool threw = false;
+
+    failing_new::failOn(1);
+    try
+    {
+        cppf::TextBuffer value("value");
+    }
+    catch (const std::bad_alloc &)
+    {
+        threw = true;
+    }
+    failing_new::disableFailure();
+    check(threw);
+    check(failing_new::liveBlocks() == baseline);
+}
+
+void testCopyAndAssignmentFailure()
+{
+    cppf::TextBuffer source("source");
+    cppf::TextBuffer destination("destination");
+    const cppf::TextBuffer &self_alias = source;
+    const std::size_t baseline = failing_new::liveBlocks();
+    bool threw = false;
+
+    failing_new::failOn(1);
+    try
+    {
+        cppf::TextBuffer copy(source);
+    }
+    catch (const std::bad_alloc &)
+    {
+        threw = true;
+    }
+    failing_new::disableFailure();
+    check(threw);
+    check(std::strcmp(source.c_str(), "source") == 0);
+    check(failing_new::liveBlocks() == baseline);
+
+    threw = false;
+    failing_new::failOn(1);
+    try
+    {
+        destination = source;
+    }
+    catch (const std::bad_alloc &)
+    {
+        threw = true;
+    }
+    failing_new::disableFailure();
+    check(threw);
+    check(std::strcmp(destination.c_str(), "destination") == 0);
+    check(failing_new::liveBlocks() == baseline);
+
+    threw = false;
+    failing_new::failOn(1);
+    try
+    {
+        source = self_alias;
+    }
+    catch (const std::bad_alloc &)
+    {
+        threw = true;
+    }
+    failing_new::disableFailure();
+    check(threw);
+    check(std::strcmp(source.c_str(), "source") == 0);
+    check(failing_new::liveBlocks() == baseline);
+}
+
+void testCompositionFailureSweep()
+{
+    cppf::TextBuffer left("left");
+    const cppf::TextBuffer right("right");
+    std::size_t observed;
+    std::size_t index;
+
+    failing_new::resetAttempts();
+    {
+        const cppf::TextBuffer joined = left + right;
+        check(std::strcmp(joined.c_str(), "leftright") == 0);
+    }
+    observed = failing_new::attempts();
+    check(observed != 0);
+    for (index = 1; index <= observed; ++index)
+    {
+        const std::size_t baseline = failing_new::liveBlocks();
+        bool threw = false;
+
+        failing_new::failOn(index);
+        try
+        {
+            const cppf::TextBuffer joined = left + right;
+        }
+        catch (const std::bad_alloc &)
+        {
+            threw = true;
+        }
+        failing_new::disableFailure();
+        check(threw);
+        check(std::strcmp(left.c_str(), "left") == 0);
+        check(std::strcmp(right.c_str(), "right") == 0);
+        check(failing_new::liveBlocks() == baseline);
+    }
+
+    {
+        const std::size_t baseline = failing_new::liveBlocks();
+        bool threw = false;
+
+        failing_new::failOn(1);
+        try
+        {
+            left += right;
+        }
+        catch (const std::bad_alloc &)
+        {
+            threw = true;
+        }
+        failing_new::disableFailure();
+        check(threw);
+        check(std::strcmp(left.c_str(), "left") == 0);
+        check(failing_new::liveBlocks() == baseline);
+    }
+}
+
+}
+
+int main()
+{
+    testConstructionFailure();
+    testCopyAndAssignmentFailure();
+    testCompositionFailureSweep();
+    if (failures != 0)
+        return 1;
+    std::cout << checks << " failure checks passed" << std::endl;
+    return 0;
+}
diff --git a/tests/support/FailingNew.cpp b/tests/support/FailingNew.cpp
new file mode 100644
index 0000000..a9e8d3a
--- /dev/null
+++ b/tests/support/FailingNew.cpp
@@ -0,0 +1,88 @@
+#include "support/FailingNew.hpp"
+
+#include <cstdlib>
+#include <new>
+
+namespace
+{
+
+std::size_t allocation_attempts = 0;
+std::size_t failure_attempt = 0;
+std::size_t live_blocks = 0;
+
+void *allocateBlock(std::size_t size)
+{
+    void *block;
+
+    ++allocation_attempts;
+    if (failure_attempt != 0 && allocation_attempts == failure_attempt)
+        throw std::bad_alloc();
+    block = std::malloc(size == 0 ? 1 : size);
+    if (block == 0)
+        throw std::bad_alloc();
+    ++live_blocks;
+    return block;
+}
+
+void freeBlock(void *block) throw()
+{
+    if (block != 0)
+    {
+        --live_blocks;
+        std::free(block);
+    }
+}
+
+}
+
+void *operator new(std::size_t size) throw(std::bad_alloc)
+{
+    return allocateBlock(size);
+}
+
+void *operator new[](std::size_t size) throw(std::bad_alloc)
+{
+    return allocateBlock(size);
+}
+
+void operator delete(void *block) throw()
+{
+    freeBlock(block);
+}
+
+void operator delete[](void *block) throw()
+{
+    freeBlock(block);
+}
+
+namespace failing_new
+{
+
+void resetAttempts()
+{
+    allocation_attempts = 0;
+    failure_attempt = 0;
+}
+
+void failOn(std::size_t attempt)
+{
+    allocation_attempts = 0;
+    failure_attempt = attempt;
+}
+
+void disableFailure()
+{
+    failure_attempt = 0;
+}
+
+std::size_t attempts()
+{
+    return allocation_attempts;
+}
+
+std::size_t liveBlocks()
+{
+    return live_blocks;
+}
+
+}
diff --git a/tests/support/FailingNew.hpp b/tests/support/FailingNew.hpp
new file mode 100644
index 0000000..ecebc95
--- /dev/null
+++ b/tests/support/FailingNew.hpp
@@ -0,0 +1,17 @@
+#ifndef CPP_FOUNDATION_FAILING_NEW_HPP
+#define CPP_FOUNDATION_FAILING_NEW_HPP
+
+#include <cstddef>
+
+namespace failing_new
+{
+
+void resetAttempts();
+void failOn(std::size_t attempt);
+void disableFailure();
+std::size_t attempts();
+std::size_t liveBlocks();
+
+}
+
+#endif


