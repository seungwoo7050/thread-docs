# 다형 객체 복제와 소유형 포맷 파이프라인

## `feat(format): 다형적 formatter 인터페이스 정의`

diff --git a/include/cppf/Formatter.hpp b/include/cppf/Formatter.hpp
new file mode 100644
index 0000000..bc95dc3
--- /dev/null
+++ b/include/cppf/Formatter.hpp
@@ -0,0 +1,55 @@
+#ifndef CPPF_FORMATTER_HPP
+#define CPPF_FORMATTER_HPP
+
+#include "cppf/TextBuffer.hpp"
+
+namespace cppf
+{
+
+class Formatter
+{
+public:
+    virtual ~Formatter();
+
+    virtual Formatter *clone() const = 0;
+    virtual TextBuffer apply(const TextBuffer &input) const = 0;
+    virtual const char *name() const = 0;
+};
+
+class UppercaseFormatter : public Formatter
+{
+public:
+    virtual Formatter *clone() const;
+    virtual TextBuffer apply(const TextBuffer &input) const;
+    virtual const char *name() const;
+};
+
+class PrefixFormatter : public Formatter
+{
+public:
+    explicit PrefixFormatter(const TextBuffer &prefix);
+
+    virtual Formatter *clone() const;
+    virtual TextBuffer apply(const TextBuffer &input) const;
+    virtual const char *name() const;
+
+private:
+    TextBuffer prefix_;
+};
+
+class SuffixFormatter : public Formatter
+{
+public:
+    explicit SuffixFormatter(const TextBuffer &suffix);
+
+    virtual Formatter *clone() const;
+    virtual TextBuffer apply(const TextBuffer &input) const;
+    virtual const char *name() const;
+
+private:
+    TextBuffer suffix_;
+};
+
+}
+
+#endif
diff --git a/src/Formatter.cpp b/src/Formatter.cpp
new file mode 100644
index 0000000..1f2d12d
--- /dev/null
+++ b/src/Formatter.cpp
@@ -0,0 +1,73 @@
+#include "cppf/Formatter.hpp"
+
+#include <cctype>
+
+namespace cppf
+{
+
+Formatter::~Formatter()
+{
+}
+
+Formatter *UppercaseFormatter::clone() const
+{
+    return new UppercaseFormatter(*this);
+}
+
+TextBuffer UppercaseFormatter::apply(const TextBuffer &input) const
+{
+    TextBuffer output(input);
+    std::size_t index;
+
+    for (index = 0; index < output.size(); ++index)
+    {
+        const unsigned char byte = static_cast<unsigned char>(output.at(index));
+        output.at(index) = static_cast<char>(std::toupper(byte));
+    }
+    return output;
+}
+
+const char *UppercaseFormatter::name() const
+{
+    return "upper";
+}
+
+PrefixFormatter::PrefixFormatter(const TextBuffer &prefix) : prefix_(prefix)
+{
+}
+
+Formatter *PrefixFormatter::clone() const
+{
+    return new PrefixFormatter(*this);
+}
+
+TextBuffer PrefixFormatter::apply(const TextBuffer &input) const
+{
+    return prefix_ + input;
+}
+
+const char *PrefixFormatter::name() const
+{
+    return "prefix";
+}
+
+SuffixFormatter::SuffixFormatter(const TextBuffer &suffix) : suffix_(suffix)
+{
+}
+
+Formatter *SuffixFormatter::clone() const
+{
+    return new SuffixFormatter(*this);
+}
+
+TextBuffer SuffixFormatter::apply(const TextBuffer &input) const
+{
+    return input + suffix_;
+}
+
+const char *SuffixFormatter::name() const
+{
+    return "suffix";
+}
+
+}


## `test(format): 파생 formatter의 동적 호출 검증`

diff --git a/tests/test_formatter.cpp b/tests/test_formatter.cpp
new file mode 100644
index 0000000..f9df7ae
--- /dev/null
+++ b/tests/test_formatter.cpp
@@ -0,0 +1,34 @@
+#include "cppf/Formatter.hpp"
+#include "support/Test.hpp"
+
+#include <cstring>
+
+namespace
+{
+
+void checkFormatter(test_support::Suite &suite,
+                    const cppf::Formatter &formatter,
+                    const char *input,
+                    const char *expected,
+                    const char *expected_name)
+{
+    const cppf::TextBuffer result = formatter.apply(cppf::TextBuffer(input));
+
+    suite.check(std::strcmp(result.c_str(), expected) == 0,
+                "formatter virtual apply");
+    suite.check(std::strcmp(formatter.name(), expected_name) == 0,
+                "formatter virtual name");
+}
+
+}
+
+void testFormatter(test_support::Suite &suite)
+{
+    const cppf::UppercaseFormatter upper;
+    const cppf::PrefixFormatter prefix(cppf::TextBuffer("["));
+    const cppf::SuffixFormatter suffix(cppf::TextBuffer("]"));
+
+    checkFormatter(suite, upper, "Abc-9", "ABC-9", "upper");
+    checkFormatter(suite, prefix, "value", "[value", "prefix");
+    checkFormatter(suite, suffix, "value", "value]", "suffix");
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 1661753..d936a92 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -3,6 +3,7 @@
 void testContact(test_support::Suite &suite);
 void testContactBook(test_support::Suite &suite);
 void testTextBuffer(test_support::Suite &suite);
+void testFormatter(test_support::Suite &suite);
 
 int main()
 {
@@ -11,5 +12,6 @@ int main()
     testContact(suite);
     testContactBook(suite);
     testTextBuffer(suite);
+    testFormatter(suite);
     return suite.result();
 }


## `feat(format): formatter 소유 pipeline 구현`

diff --git a/include/cppf/FormatPipeline.hpp b/include/cppf/FormatPipeline.hpp
new file mode 100644
index 0000000..4a0934c
--- /dev/null
+++ b/include/cppf/FormatPipeline.hpp
@@ -0,0 +1,37 @@
+#ifndef CPPF_FORMAT_PIPELINE_HPP
+#define CPPF_FORMAT_PIPELINE_HPP
+
+#include "cppf/Formatter.hpp"
+
+#include <cstddef>
+
+namespace cppf
+{
+
+class FormatPipeline
+{
+public:
+    enum
+    {
+        max_steps = 8
+    };
+
+    FormatPipeline();
+    ~FormatPipeline();
+
+    std::size_t size() const;
+    void append(const Formatter &formatter);
+    TextBuffer apply(const TextBuffer &input) const;
+    void swap(FormatPipeline &other) throw();
+
+private:
+    FormatPipeline(const FormatPipeline &other);
+    FormatPipeline &operator=(const FormatPipeline &other);
+
+    Formatter *steps_[max_steps];
+    std::size_t size_;
+};
+
+}
+
+#endif
diff --git a/src/FormatPipeline.cpp b/src/FormatPipeline.cpp
new file mode 100644
index 0000000..2dc69f5
--- /dev/null
+++ b/src/FormatPipeline.cpp
@@ -0,0 +1,65 @@
+#include "cppf/FormatPipeline.hpp"
+
+#include <stdexcept>
+
+namespace cppf
+{
+
+FormatPipeline::FormatPipeline() : steps_(), size_(0)
+{
+    std::size_t index;
+
+    for (index = 0; index < max_steps; ++index)
+        steps_[index] = 0;
+}
+
+FormatPipeline::~FormatPipeline()
+{
+    std::size_t index;
+
+    for (index = 0; index < size_; ++index)
+        delete steps_[index];
+}
+
+std::size_t FormatPipeline::size() const
+{
+    return size_;
+}
+
+void FormatPipeline::append(const Formatter &formatter)
+{
+    Formatter *copy;
+
+    if (size_ == max_steps)
+        throw std::length_error("pipeline capacity");
+    copy = formatter.clone();
+    steps_[size_] = copy;
+    ++size_;
+}
+
+TextBuffer FormatPipeline::apply(const TextBuffer &input) const
+{
+    TextBuffer result(input);
+    std::size_t index;
+
+    for (index = 0; index < size_; ++index)
+        result = steps_[index]->apply(result);
+    return result;
+}
+
+void FormatPipeline::swap(FormatPipeline &other) throw()
+{
+    std::size_t index;
+    const std::size_t size = size_;
+
+    for (index = 0; index < max_steps; ++index)
+    {
+        Formatter *step = steps_[index];
+        steps_[index] = other.steps_[index];
+        other.steps_[index] = step;
+    }
+    size_ = other.size_;
+    other.size_ = size;
+}
+
+}


## `feat(format): pipeline 실행 CLI 제공`

diff --git a/apps/ex02_format_pipeline.cpp b/apps/ex02_format_pipeline.cpp
new file mode 100644
index 0000000..a4b7985
--- /dev/null
+++ b/apps/ex02_format_pipeline.cpp
@@ -0,0 +1,22 @@
+#include "cppf/FormatPipeline.hpp"
+
+#include <iostream>
+
+int main(int argument_count, char **arguments)
+{
+    if (argument_count != 2)
+    {
+        std::cerr << "usage: ex02_format_pipeline TEXT" << std::endl;
+        return 1;
+    }
+    const cppf::PrefixFormatter prefix(cppf::TextBuffer("["));
+    const cppf::UppercaseFormatter upper;
+    const cppf::SuffixFormatter suffix(cppf::TextBuffer("]"));
+    cppf::FormatPipeline pipeline;
+
+    pipeline.append(prefix);
+    pipeline.append(upper);
+    pipeline.append(suffix);
+    std::cout << pipeline.apply(cppf::TextBuffer(arguments[1])) << std::endl;
+    return 0;
+}


## `test(format): pipeline 적용 순서와 용량 경계 검증`

diff --git a/tests/test_format_pipeline.cpp b/tests/test_format_pipeline.cpp
new file mode 100644
index 0000000..27fc1b0
--- /dev/null
+++ b/tests/test_format_pipeline.cpp
@@ -0,0 +1,42 @@
+#include "cppf/FormatPipeline.hpp"
+#include "support/Test.hpp"
+
+#include <cstring>
+#include <stdexcept>
+
+void testFormatPipeline(test_support::Suite &suite)
+{
+    cppf::FormatPipeline pipeline;
+    const cppf::PrefixFormatter prefix(cppf::TextBuffer("["));
+    const cppf::UppercaseFormatter upper;
+    const cppf::SuffixFormatter suffix(cppf::TextBuffer("]"));
+    bool threw = false;
+    std::size_t index;
+
+    suite.check(pipeline.size() == 0, "format pipeline starts empty");
+    suite.check(pipeline.apply(cppf::TextBuffer("same")) ==
+                    cppf::TextBuffer("same"),
+                "empty format pipeline is identity");
+    pipeline.append(prefix);
+    pipeline.append(upper);
+    pipeline.append(suffix);
+    suite.check(pipeline.size() == 3, "format pipeline counts clones");
+    suite.check(pipeline.apply(cppf::TextBuffer("value")) ==
+                    cppf::TextBuffer("[VALUE]"),
+                "format pipeline dispatch order");
+
+    cppf::FormatPipeline full;
+    for (index = 0; index < cppf::FormatPipeline::max_steps; ++index)
+        full.append(upper);
+    try
+    {
+        full.append(upper);
+    }
+    catch (const std::length_error &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "format pipeline rejects capacity overflow");
+    suite.check(full.size() == cppf::FormatPipeline::max_steps,
+                "capacity failure preserves pipeline");
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index d936a92..912945b 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -4,6 +4,7 @@ void testContact(test_support::Suite &suite);
 void testContactBook(test_support::Suite &suite);
 void testTextBuffer(test_support::Suite &suite);
 void testFormatter(test_support::Suite &suite);
+void testFormatPipeline(test_support::Suite &suite);
 
 int main()
 {
@@ -13,5 +14,6 @@ int main()
     testContactBook(suite);
     testTextBuffer(suite);
     testFormatter(suite);
+    testFormatPipeline(suite);
     return suite.result();
 }


## `feat(format): pipeline 깊은 복사 구현`

diff --git a/include/cppf/FormatPipeline.hpp b/include/cppf/FormatPipeline.hpp
index 4a0934c..42ef08a 100644
--- a/include/cppf/FormatPipeline.hpp
+++ b/include/cppf/FormatPipeline.hpp
@@ -17,7 +17,9 @@ public:
     };
 
     FormatPipeline();
+    FormatPipeline(const FormatPipeline &other);
     ~FormatPipeline();
+    FormatPipeline &operator=(const FormatPipeline &other);
 
     std::size_t size() const;
     void append(const Formatter &formatter);
@@ -25,9 +27,6 @@ public:
     void swap(FormatPipeline &other) throw();
 
 private:
-    FormatPipeline(const FormatPipeline &other);
-    FormatPipeline &operator=(const FormatPipeline &other);
-
     Formatter *steps_[max_steps];
     std::size_t size_;
 };
diff --git a/src/FormatPipeline.cpp b/src/FormatPipeline.cpp
index 2dc69f5..a157a87 100644
--- a/src/FormatPipeline.cpp
+++ b/src/FormatPipeline.cpp
@@ -13,6 +13,25 @@ FormatPipeline::FormatPipeline() : steps_(), size_(0)
         steps_[index] = 0;
 }
 
+FormatPipeline::FormatPipeline(const FormatPipeline &other) : steps_(), size_(0)
+{
+    std::size_t index;
+
+    for (index = 0; index < max_steps; ++index)
+        steps_[index] = 0;
+    try
+    {
+        for (index = 0; index < other.size_; ++index)
+            append(*other.steps_[index]);
+    }
+    catch (...)
+    {
+        for (index = 0; index < size_; ++index)
+            delete steps_[index];
+        throw;
+    }
+}
+
 FormatPipeline::~FormatPipeline()
 {
     std::size_t index;
@@ -21,6 +40,14 @@ FormatPipeline::~FormatPipeline()
         delete steps_[index];
 }
 
+FormatPipeline &FormatPipeline::operator=(const FormatPipeline &other)
+{
+    FormatPipeline copy(other);
+
+    swap(copy);
+    return *this;
+}
+
 std::size_t FormatPipeline::size() const
 {
     return size_;


## `test(format): pipeline 복사와 자기 대입 검증`

diff --git a/tests/test_format_pipeline.cpp b/tests/test_format_pipeline.cpp
index 27fc1b0..e118617 100644
--- a/tests/test_format_pipeline.cpp
+++ b/tests/test_format_pipeline.cpp
@@ -25,6 +25,24 @@ void testFormatPipeline(test_support::Suite &suite)
                     cppf::TextBuffer("[VALUE]"),
                 "format pipeline dispatch order");
 
+    cppf::FormatPipeline copy(pipeline);
+    cppf::FormatPipeline assigned;
+    const cppf::FormatPipeline &self_alias = pipeline;
+
+    pipeline.append(suffix);
+    suite.check(copy.size() == 3, "format pipeline copy owns independent steps");
+    suite.check(copy.apply(cppf::TextBuffer("value")) ==
+                    cppf::TextBuffer("[VALUE]"),
+                "format pipeline copy preserves dynamic behavior");
+    suite.check(&(assigned = copy) == &assigned,
+                "format pipeline assignment returns self");
+    suite.check(assigned.apply(cppf::TextBuffer("x")) ==
+                    cppf::TextBuffer("[X]"),
+                "format pipeline assignment clones steps");
+    pipeline = self_alias;
+    suite.check(pipeline.size() == 4,
+                "format pipeline self assignment preserves state");
+
     cppf::FormatPipeline full;
     for (index = 0; index < cppf::FormatPipeline::max_steps; ++index)
         full.append(upper);


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


