# 공개 헤더와 컴파일 시간 사용 계약

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


## `test(format): 가상 소멸·추상 계약·CLI 검증`

diff --git a/Makefile b/Makefile
index 5010050..8ad993c 100644
--- a/Makefile
+++ b/Makefile
@@ -20,6 +20,7 @@ APP_SRC := $(sort $(wildcard apps/*.cpp))
 APP_BIN := $(APP_SRC:apps/%.cpp=bin/%)
 
 TEST_SRC := $(sort $(wildcard tests/test_*.cpp))
+TEST_SUPPORT_SRC := tests/support/TestFormatter.cpp
 TEST_BIN := build/tests/unit
 FAILURE_BIN := build/tests/buffer_failure
 FAILURE_SRC := tests/failure/test_buffer_failure.cpp \
@@ -43,9 +44,10 @@ bin/%: apps/%.cpp $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $< $(NAME) -o $@
 
-$(TEST_BIN): $(TEST_SRC) $(NAME)
+$(TEST_BIN): $(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(TEST_SRC) $(NAME) -o $@
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(TEST_SRC) $(TEST_SUPPORT_SRC) \
+		$(NAME) -o $@
 
 test-unit: $(TEST_BIN)
 	./$(TEST_BIN)
@@ -57,10 +59,10 @@ $(FAILURE_BIN): $(FAILURE_SRC) $(NAME)
 failure-test: $(FAILURE_BIN)
 	./$(FAILURE_BIN)
 
-$(NO_ELIDE_BIN): $(TEST_SRC) $(NAME)
+$(NO_ELIDE_BIN): $(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fno-elide-constructors \
-		$(TEST_SRC) $(NAME) -o $@
+		$(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME) -o $@
 
 test-no-elide: $(NO_ELIDE_BIN)
 	./$(NO_ELIDE_BIN)
@@ -68,8 +70,12 @@ test-no-elide: $(NO_ELIDE_BIN)
 test-contract:
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/format_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/formatter_abstract_fail.cpp >/dev/null 2>&1
 
 test-integration: bin/ex00_contact_book
 	sh tests/check_cli.sh
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index d115730..8ec9cab 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -12,3 +12,7 @@ diff -u tests/fixtures/contact-session.out "$temporary_directory/contact.out"
 ./bin/ex01_text_buffer hello world > "$temporary_directory/text.out"
 printf 'helloworld\n' > "$temporary_directory/text.expected"
 diff -u "$temporary_directory/text.expected" "$temporary_directory/text.out"
+
+./bin/ex02_format_pipeline mixed > "$temporary_directory/format.out"
+printf '[MIXED]\n' > "$temporary_directory/format.expected"
+diff -u "$temporary_directory/format.expected" "$temporary_directory/format.out"
diff --git a/tests/compile/format_headers.cpp b/tests/compile/format_headers.cpp
new file mode 100644
index 0000000..152f0e1
--- /dev/null
+++ b/tests/compile/format_headers.cpp
@@ -0,0 +1,15 @@
+#include "cppf/Formatter.hpp"
+#include "cppf/Formatter.hpp"
+#include "cppf/FormatPipeline.hpp"
+#include "cppf/FormatPipeline.hpp"
+
+int main()
+{
+    const cppf::UppercaseFormatter formatter;
+    cppf::FormatPipeline pipeline;
+
+    pipeline.append(formatter);
+    return pipeline.apply(cppf::TextBuffer("a")) == cppf::TextBuffer("A")
+               ? 0
+               : 1;
+}
diff --git a/tests/compile/formatter_abstract_fail.cpp b/tests/compile/formatter_abstract_fail.cpp
new file mode 100644
index 0000000..dc10514
--- /dev/null
+++ b/tests/compile/formatter_abstract_fail.cpp
@@ -0,0 +1,8 @@
+#include "cppf/Formatter.hpp"
+
+int main()
+{
+    cppf::Formatter formatter;
+
+    return formatter.name()[0];
+}
diff --git a/tests/support/TestFormatter.cpp b/tests/support/TestFormatter.cpp
new file mode 100644
index 0000000..fabf4ba
--- /dev/null
+++ b/tests/support/TestFormatter.cpp
@@ -0,0 +1,57 @@
+#include "support/TestFormatter.hpp"
+
+namespace test_support
+{
+
+int TestFormatter::live_count_ = 0;
+int TestFormatter::destroyed_count_ = 0;
+
+TestFormatter::TestFormatter(const cppf::TextBuffer &prefix) : prefix_(prefix)
+{
+    ++live_count_;
+}
+
+TestFormatter::TestFormatter(const TestFormatter &other)
+    : cppf::Formatter(other), prefix_(other.prefix_)
+{
+    ++live_count_;
+}
+
+TestFormatter::~TestFormatter()
+{
+    --live_count_;
+    ++destroyed_count_;
+}
+
+cppf::Formatter *TestFormatter::clone() const
+{
+    return new TestFormatter(*this);
+}
+
+cppf::TextBuffer TestFormatter::apply(const cppf::TextBuffer &input) const
+{
+    return prefix_ + input;
+}
+
+const char *TestFormatter::name() const
+{
+    return "test";
+}
+
+void TestFormatter::resetCounters()
+{
+    live_count_ = 0;
+    destroyed_count_ = 0;
+}
+
+int TestFormatter::liveCount()
+{
+    return live_count_;
+}
+
+int TestFormatter::destroyedCount()
+{
+    return destroyed_count_;
+}
+
+}
diff --git a/tests/support/TestFormatter.hpp b/tests/support/TestFormatter.hpp
new file mode 100644
index 0000000..f0dd639
--- /dev/null
+++ b/tests/support/TestFormatter.hpp
@@ -0,0 +1,34 @@
+#ifndef CPP_FOUNDATION_TEST_FORMATTER_HPP
+#define CPP_FOUNDATION_TEST_FORMATTER_HPP
+
+#include "cppf/Formatter.hpp"
+
+namespace test_support
+{
+
+class TestFormatter : public cppf::Formatter
+{
+public:
+    explicit TestFormatter(const cppf::TextBuffer &prefix);
+    TestFormatter(const TestFormatter &other);
+    virtual ~TestFormatter();
+
+    virtual cppf::Formatter *clone() const;
+    virtual cppf::TextBuffer apply(const cppf::TextBuffer &input) const;
+    virtual const char *name() const;
+
+    static void resetCounters();
+    static int liveCount();
+    static int destroyedCount();
+
+private:
+    TestFormatter &operator=(const TestFormatter &other);
+
+    cppf::TextBuffer prefix_;
+    static int live_count_;
+    static int destroyed_count_;
+};
+
+}
+
+#endif
diff --git a/tests/test_formatter.cpp b/tests/test_formatter.cpp
index f9df7ae..162d65b 100644
--- a/tests/test_formatter.cpp
+++ b/tests/test_formatter.cpp
@@ -1,5 +1,6 @@
 #include "cppf/Formatter.hpp"
 #include "support/Test.hpp"
+#include "support/TestFormatter.hpp"
 
 #include <cstring>
 
@@ -31,4 +32,24 @@ void testFormatter(test_support::Suite &suite)
     checkFormatter(suite, upper, "Abc-9", "ABC-9", "upper");
     checkFormatter(suite, prefix, "value", "[value", "prefix");
     checkFormatter(suite, suffix, "value", "value]", "suffix");
+
+    test_support::TestFormatter::resetCounters();
+    cppf::Formatter *owned =
+        new test_support::TestFormatter(cppf::TextBuffer("!"));
+    suite.check(test_support::TestFormatter::liveCount() == 1,
+                "derived formatter becomes live");
+    delete owned;
+    suite.check(test_support::TestFormatter::liveCount() == 0,
+                "base deletion destroys derived formatter");
+    suite.check(test_support::TestFormatter::destroyedCount() == 1,
+                "derived destructor runs exactly once");
+
+    const test_support::TestFormatter original(cppf::TextBuffer("#"));
+    cppf::Formatter *cloned = original.clone();
+    suite.check(cloned->apply(cppf::TextBuffer("x")) ==
+                    cppf::TextBuffer("#x"),
+                "polymorphic clone preserves dynamic state");
+    suite.check(std::strcmp(cloned->name(), "test") == 0,
+                "polymorphic clone preserves dynamic name");
+    delete cloned;
 }


## `test(casts): 타입·주소 변환의 공개 경계 검증`

diff --git a/Makefile b/Makefile
index 4ca9b2f..a29f156 100644
--- a/Makefile
+++ b/Makefile
@@ -90,6 +90,14 @@ test-contract:
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/formatter_abstract_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_inspector_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_unrelated_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/serializer_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/serializer_const_fail.cpp >/dev/null 2>&1
 
 test-integration: $(APP_BIN)
 	sh tests/check_cli.sh
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index 18d08ba..6069752 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -52,3 +52,51 @@ printf 'invalid scalar literal\n' \
     > "$temporary_directory/scalar-failure.expected"
 diff -u "$temporary_directory/scalar-failure.expected" \
     "$temporary_directory/scalar-failure.err"
+
+./bin/ex04_type_boundary runtime A \
+    > "$temporary_directory/runtime.out"
+printf 'pointer: A\nreference: A\n' \
+    > "$temporary_directory/runtime.expected"
+diff -u "$temporary_directory/runtime.expected" \
+    "$temporary_directory/runtime.out"
+
+./bin/ex04_type_boundary address 42 alpha \
+    > "$temporary_directory/address.out"
+printf 'token: nonzero\nsame: yes\nid: 42\nlabel: alpha\n' \
+    > "$temporary_directory/address.expected"
+diff -u "$temporary_directory/address.expected" \
+    "$temporary_directory/address.out"
+
+if ./bin/ex04_type_boundary runtime Z \
+    > "$temporary_directory/runtime-failure.out" \
+    2> "$temporary_directory/runtime-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/runtime-failure.out"
+printf 'unknown runtime kind\n' \
+    > "$temporary_directory/runtime-failure.expected"
+diff -u "$temporary_directory/runtime-failure.expected" \
+    "$temporary_directory/runtime-failure.err"
+
+if ./bin/ex04_type_boundary address 42x alpha \
+    > "$temporary_directory/address-failure.out" \
+    2> "$temporary_directory/address-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/address-failure.out"
+printf 'invalid payload id\n' \
+    > "$temporary_directory/address-failure.expected"
+diff -u "$temporary_directory/address-failure.expected" \
+    "$temporary_directory/address-failure.err"
+
+if ./bin/ex04_type_boundary address 18446744073709551616 alpha \
+    > "$temporary_directory/address-overflow.out" \
+    2> "$temporary_directory/address-overflow.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/address-overflow.out"
+diff -u "$temporary_directory/address-failure.expected" \
+    "$temporary_directory/address-overflow.err"
diff --git a/tests/compile/runtime_inspector_private_fail.cpp b/tests/compile/runtime_inspector_private_fail.cpp
new file mode 100644
index 0000000..a142f7e
--- /dev/null
+++ b/tests/compile/runtime_inspector_private_fail.cpp
@@ -0,0 +1,9 @@
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    cppf::RuntimeInspector inspector;
+    return cppf::RuntimeInspector::identify(
+               static_cast<const cppf::RuntimeBase *>(0)) ==
+           cppf::runtime_unknown;
+}
diff --git a/tests/compile/runtime_unrelated_fail.cpp b/tests/compile/runtime_unrelated_fail.cpp
new file mode 100644
index 0000000..0ec158a
--- /dev/null
+++ b/tests/compile/runtime_unrelated_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    int value = 0;
+    return cppf::RuntimeInspector::identify(&value);
+}
diff --git a/tests/compile/serializer_const_fail.cpp b/tests/compile/serializer_const_fail.cpp
new file mode 100644
index 0000000..251488f
--- /dev/null
+++ b/tests/compile/serializer_const_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/Serializer.hpp"
+
+int main()
+{
+    const cppf::Payload payload(7, "value");
+    return cppf::Serializer::serialize(&payload) == 0;
+}
diff --git a/tests/compile/serializer_private_fail.cpp b/tests/compile/serializer_private_fail.cpp
new file mode 100644
index 0000000..1e8d2ab
--- /dev/null
+++ b/tests/compile/serializer_private_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/Serializer.hpp"
+
+int main()
+{
+    cppf::Serializer serializer;
+    return 0;
+}
diff --git a/tests/test_runtime_type.cpp b/tests/test_runtime_type.cpp
index 14d6ae5..dde3035 100644
--- a/tests/test_runtime_type.cpp
+++ b/tests/test_runtime_type.cpp
@@ -30,6 +30,10 @@ private:
     bool &destroyed_;
 };
 
+class KnownChild : public cppf::RuntimeA
+{
+};
+
 }
 
 void testRuntimeType(test_support::Suite &suite)
@@ -83,6 +87,12 @@ void testRuntimeType(test_support::Suite &suite)
     suite.check(created, "runtime factory creates every registered type");
     suite.check(cppf::RuntimeInspector::create(cppf::runtime_unknown) == 0,
                 "runtime factory rejects unknown kind");
+    const cppf::RuntimeKind invalid_kind =
+        static_cast<cppf::RuntimeKind>(999);
+    suite.check(cppf::RuntimeInspector::create(invalid_kind) == 0 &&
+                    std::strcmp(cppf::RuntimeInspector::name(invalid_kind),
+                                "unknown") == 0,
+                "runtime rejects invalid enum value");
     suite.check(std::strcmp(cppf::RuntimeInspector::name(cppf::runtime_a),
                             "A") == 0 &&
                     std::strcmp(
@@ -90,6 +100,24 @@ void testRuntimeType(test_support::Suite &suite)
                         "unknown") == 0,
                 "runtime names are stable");
 
+    const KnownChild known_child;
+    suite.check(cppf::RuntimeInspector::identify(&known_child) ==
+                    cppf::runtime_a &&
+                    cppf::RuntimeInspector::identify(known_child) ==
+                    cppf::runtime_a,
+                "runtime recognizes a registered subtype descendant");
+
+    bool escaped = false;
+    try
+    {
+        cppf::RuntimeInspector::identify(unknown);
+    }
+    catch (...)
+    {
+        escaped = true;
+    }
+    suite.check(!escaped, "runtime reference hides bad cast failures");
+
     bool destroyed = false;
     cppf::RuntimeBase *tracked = new TrackedRuntime(destroyed);
 


