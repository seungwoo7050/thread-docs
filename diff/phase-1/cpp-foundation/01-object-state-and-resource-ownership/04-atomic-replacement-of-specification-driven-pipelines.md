# 문자열 명세 기반 파이프라인의 원자 교체

## `feat(factory): 문자열 명세로 formatter 생성`

diff --git a/include/cppf/Factory.hpp b/include/cppf/Factory.hpp
new file mode 100644
index 0000000..f1cee2e
--- /dev/null
+++ b/include/cppf/Factory.hpp
@@ -0,0 +1,52 @@
+#ifndef CPPF_FACTORY_HPP
+#define CPPF_FACTORY_HPP
+
+#include "cppf/FormatPipeline.hpp"
+
+#include <cstddef>
+#include <exception>
+#include <string>
+
+namespace cppf
+{
+
+class InvalidSpecification : public std::exception
+{
+public:
+    virtual const char *what() const throw();
+};
+
+class UnknownFormatter : public std::exception
+{
+public:
+    virtual const char *what() const throw();
+};
+
+class FormatterCreator
+{
+public:
+    virtual ~FormatterCreator();
+    virtual Formatter *create(const std::string &specification) const = 0;
+};
+
+class DefaultFormatterCreator : public FormatterCreator
+{
+public:
+    virtual Formatter *create(const std::string &specification) const;
+};
+
+class PipelineBuilder
+{
+public:
+    static void replace(FormatPipeline &target,
+                        const FormatterCreator &creator,
+                        const std::string *specifications,
+                        std::size_t count);
+
+private:
+    PipelineBuilder();
+};
+
+}
+
+#endif
diff --git a/src/Factory.cpp b/src/Factory.cpp
new file mode 100644
index 0000000..b933f53
--- /dev/null
+++ b/src/Factory.cpp
@@ -0,0 +1,47 @@
+#include "cppf/Factory.hpp"
+
+namespace cppf
+{
+
+const char *InvalidSpecification::what() const throw()
+{
+    return "invalid formatter specification";
+}
+
+const char *UnknownFormatter::what() const throw()
+{
+    return "unknown formatter";
+}
+
+FormatterCreator::~FormatterCreator()
+{
+}
+
+Formatter *DefaultFormatterCreator::create(
+    const std::string &specification) const
+{
+    const std::string prefix_key = "prefix=";
+    const std::string suffix_key = "suffix=";
+
+    if (specification.empty())
+        throw InvalidSpecification();
+    if (specification == "upper")
+        return new UppercaseFormatter();
+    if (specification.compare(0, prefix_key.size(), prefix_key) == 0)
+    {
+        if (specification.size() == prefix_key.size())
+            throw InvalidSpecification();
+        return new PrefixFormatter(
+            TextBuffer(specification.substr(prefix_key.size()).c_str()));
+    }
+    if (specification.compare(0, suffix_key.size(), suffix_key) == 0)
+    {
+        if (specification.size() == suffix_key.size())
+            throw InvalidSpecification();
+        return new SuffixFormatter(
+            TextBuffer(specification.substr(suffix_key.size()).c_str()));
+    }
+    throw UnknownFormatter();
+}
+
+}


## `test(factory): formatter 명세 분류 검증`

diff --git a/tests/test_factory.cpp b/tests/test_factory.cpp
new file mode 100644
index 0000000..22e9f29
--- /dev/null
+++ b/tests/test_factory.cpp
@@ -0,0 +1,61 @@
+#include "cppf/Factory.hpp"
+#include "support/Test.hpp"
+
+#include <cstring>
+
+void testFactory(test_support::Suite &suite)
+{
+    const cppf::DefaultFormatterCreator creator;
+    cppf::Formatter *formatter;
+    bool threw;
+
+    formatter = creator.create("upper");
+    suite.check(std::strcmp(formatter->name(), "upper") == 0,
+                "factory creates uppercase formatter");
+    delete formatter;
+
+    formatter = creator.create("prefix=[");
+    suite.check(formatter->apply(cppf::TextBuffer("x")) ==
+                    cppf::TextBuffer("[x"),
+                "factory parses prefix payload");
+    delete formatter;
+
+    formatter = creator.create("suffix=]");
+    suite.check(formatter->apply(cppf::TextBuffer("x")) ==
+                    cppf::TextBuffer("x]"),
+                "factory parses suffix payload");
+    delete formatter;
+
+    threw = false;
+    try
+    {
+        formatter = creator.create("");
+    }
+    catch (const cppf::InvalidSpecification &error)
+    {
+        threw = std::strcmp(error.what(), "invalid formatter specification") == 0;
+    }
+    suite.check(threw, "factory rejects empty specification");
+
+    threw = false;
+    try
+    {
+        formatter = creator.create("prefix=");
+    }
+    catch (const cppf::InvalidSpecification &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "factory rejects empty payload");
+
+    threw = false;
+    try
+    {
+        formatter = creator.create("reverse");
+    }
+    catch (const cppf::UnknownFormatter &error)
+    {
+        threw = std::strcmp(error.what(), "unknown formatter") == 0;
+    }
+    suite.check(threw, "factory distinguishes unknown formatter");
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 912945b..0183fcf 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -5,6 +5,7 @@ void testContactBook(test_support::Suite &suite);
 void testTextBuffer(test_support::Suite &suite);
 void testFormatter(test_support::Suite &suite);
 void testFormatPipeline(test_support::Suite &suite);
+void testFactory(test_support::Suite &suite);
 
 int main()
 {
@@ -15,5 +16,6 @@ int main()
     testTextBuffer(suite);
     testFormatter(suite);
     testFormatPipeline(suite);
+    testFactory(suite);
     return suite.result();
 }


## `feat(factory): formatter 임시 소유와 pipeline 교체 구현`

diff --git a/src/Factory.cpp b/src/Factory.cpp
index b933f53..cf52b41 100644
--- a/src/Factory.cpp
+++ b/src/Factory.cpp
@@ -1,5 +1,34 @@
 #include "cppf/Factory.hpp"
 
+namespace
+{
+
+class FormatterOwner
+{
+public:
+    explicit FormatterOwner(cppf::Formatter *formatter) : formatter_(formatter)
+    {
+    }
+
+    ~FormatterOwner()
+    {
+        delete formatter_;
+    }
+
+    cppf::Formatter &get() const
+    {
+        return *formatter_;
+    }
+
+private:
+    FormatterOwner(const FormatterOwner &other);
+    FormatterOwner &operator=(const FormatterOwner &other);
+
+    cppf::Formatter *formatter_;
+};
+
+}
+
 namespace cppf
 {
 
@@ -44,4 +73,23 @@ Formatter *DefaultFormatterCreator::create(
     throw UnknownFormatter();
 }
 
+void PipelineBuilder::replace(FormatPipeline &target,
+                              const FormatterCreator &creator,
+                              const std::string *specifications,
+                              std::size_t count)
+{
+    FormatPipeline empty;
+    std::size_t index;
+
+    if ((specifications == 0 && count != 0) ||
+        count > FormatPipeline::max_steps)
+        throw InvalidSpecification();
+    target.swap(empty);
+    for (index = 0; index < count; ++index)
+    {
+        FormatterOwner formatter(creator.create(specifications[index]));
+        target.append(formatter.get());
+    }
+}
+
 }


## `test(factory): builder 소유권 이전 검증`

diff --git a/tests/test_factory.cpp b/tests/test_factory.cpp
index 22e9f29..27ab61d 100644
--- a/tests/test_factory.cpp
+++ b/tests/test_factory.cpp
@@ -58,4 +58,27 @@ void testFactory(test_support::Suite &suite)
         threw = std::strcmp(error.what(), "unknown formatter") == 0;
     }
     suite.check(threw, "factory distinguishes unknown formatter");
+
+    std::string specifications[] = {"prefix=[", "upper", "suffix=]"};
+    cppf::FormatPipeline pipeline;
+
+    cppf::PipelineBuilder::replace(pipeline, creator, specifications, 3);
+    suite.check(pipeline.size() == 3, "pipeline builder transfers three clones");
+    suite.check(pipeline.apply(cppf::TextBuffer("value")) ==
+                    cppf::TextBuffer("[VALUE]"),
+                "pipeline builder preserves specification order");
+
+    cppf::PipelineBuilder::replace(pipeline, creator, 0, 0);
+    suite.check(pipeline.size() == 0, "pipeline builder accepts empty list");
+
+    threw = false;
+    try
+    {
+        cppf::PipelineBuilder::replace(pipeline, creator, 0, 1);
+    }
+    catch (const cppf::InvalidSpecification &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "pipeline builder rejects null specification array");
 }


## `fix(factory): 교체 실패에도 기존 파이프라인 보존`

diff --git a/src/Factory.cpp b/src/Factory.cpp
index cf52b41..9b98a98 100644
--- a/src/Factory.cpp
+++ b/src/Factory.cpp
@@ -78,18 +78,18 @@ void PipelineBuilder::replace(FormatPipeline &target,
                               const std::string *specifications,
                               std::size_t count)
 {
-    FormatPipeline empty;
+    FormatPipeline candidate;
     std::size_t index;
 
     if ((specifications == 0 && count != 0) ||
         count > FormatPipeline::max_steps)
         throw InvalidSpecification();
-    target.swap(empty);
     for (index = 0; index < count; ++index)
     {
         FormatterOwner formatter(creator.create(specifications[index]));
-        target.append(formatter.get());
+        candidate.append(formatter.get());
     }
+    target.swap(candidate);
 }
 
 }


## `feat(factory): 명세 기반 파이프라인 CLI 제공`

diff --git a/apps/ex03_pipeline_factory.cpp b/apps/ex03_pipeline_factory.cpp
new file mode 100644
index 0000000..e4cac3d
--- /dev/null
+++ b/apps/ex03_pipeline_factory.cpp
@@ -0,0 +1,35 @@
+#include "cppf/Factory.hpp"
+
+#include <exception>
+#include <iostream>
+#include <string>
+
+int main(int argument_count, char **arguments)
+{
+    std::string specifications[cppf::FormatPipeline::max_steps];
+    cppf::FormatPipeline pipeline;
+    const cppf::DefaultFormatterCreator creator;
+    int index;
+
+    if (argument_count < 3 ||
+        argument_count - 2 > cppf::FormatPipeline::max_steps)
+    {
+        std::cerr << "usage: ex03_pipeline_factory TEXT SPEC..." << std::endl;
+        return 1;
+    }
+    for (index = 2; index < argument_count; ++index)
+        specifications[index - 2] = arguments[index];
+    try
+    {
+        cppf::PipelineBuilder::replace(
+            pipeline, creator, specifications, argument_count - 2);
+        std::cout << pipeline.apply(cppf::TextBuffer(arguments[1]))
+                  << std::endl;
+    }
+    catch (const std::exception &error)
+    {
+        std::cerr << error.what() << std::endl;
+        return 1;
+    }
+    return 0;
+}


## `test(factory): 교체 실패 상태 보존과 CLI 검증`

diff --git a/Makefile b/Makefile
index 8ad993c..034530c 100644
--- a/Makefile
+++ b/Makefile
@@ -77,7 +77,7 @@ test-contract:
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/formatter_abstract_fail.cpp >/dev/null 2>&1
 
-test-integration: bin/ex00_contact_book
+test-integration: $(APP_BIN)
 	sh tests/check_cli.sh
 
 test: test-unit failure-test test-no-elide test-contract test-integration
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index 8ec9cab..56973df 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -16,3 +16,20 @@ diff -u "$temporary_directory/text.expected" "$temporary_directory/text.out"
 ./bin/ex02_format_pipeline mixed > "$temporary_directory/format.out"
 printf '[MIXED]\n' > "$temporary_directory/format.expected"
 diff -u "$temporary_directory/format.expected" "$temporary_directory/format.out"
+
+./bin/ex03_pipeline_factory mixed 'prefix=[' upper 'suffix=]' \
+    > "$temporary_directory/factory.out"
+printf '[MIXED]\n' > "$temporary_directory/factory.expected"
+diff -u "$temporary_directory/factory.expected" \
+    "$temporary_directory/factory.out"
+
+if ./bin/ex03_pipeline_factory mixed reverse \
+    > "$temporary_directory/factory-failure.out" \
+    2> "$temporary_directory/factory-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/factory-failure.out"
+printf 'unknown formatter\n' > "$temporary_directory/factory-failure.expected"
+diff -u "$temporary_directory/factory-failure.expected" \
+    "$temporary_directory/factory-failure.err"
diff --git a/tests/test_factory.cpp b/tests/test_factory.cpp
index 27ab61d..25022ba 100644
--- a/tests/test_factory.cpp
+++ b/tests/test_factory.cpp
@@ -68,8 +68,25 @@ void testFactory(test_support::Suite &suite)
                     cppf::TextBuffer("[VALUE]"),
                 "pipeline builder preserves specification order");
 
-    cppf::PipelineBuilder::replace(pipeline, creator, 0, 0);
-    suite.check(pipeline.size() == 0, "pipeline builder accepts empty list");
+    std::string invalid_specifications[] = {
+        "prefix=<", "reverse", "suffix=>"};
+
+    threw = false;
+    try
+    {
+        cppf::PipelineBuilder::replace(
+            pipeline, creator, invalid_specifications, 3);
+    }
+    catch (const cppf::UnknownFormatter &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "pipeline builder reports a failed replacement");
+    suite.check(pipeline.size() == 3,
+                "failed replacement preserves the previous pipeline size");
+    suite.check(pipeline.apply(cppf::TextBuffer("value")) ==
+                    cppf::TextBuffer("[VALUE]"),
+                "failed replacement preserves the previous pipeline value");
 
     threw = false;
     try
@@ -81,4 +98,10 @@ void testFactory(test_support::Suite &suite)
         threw = true;
     }
     suite.check(threw, "pipeline builder rejects null specification array");
+    suite.check(pipeline.apply(cppf::TextBuffer("value")) ==
+                    cppf::TextBuffer("[VALUE]"),
+                "invalid replacement preserves the previous pipeline");
+
+    cppf::PipelineBuilder::replace(pipeline, creator, 0, 0);
+    suite.check(pipeline.size() == 0, "pipeline builder accepts empty list");
 }


## `test(factory): 생성·복제·할당 실패 정리 검증`

diff --git a/Makefile b/Makefile
index 034530c..47551ad 100644
--- a/Makefile
+++ b/Makefile
@@ -25,6 +25,9 @@ TEST_BIN := build/tests/unit
 FAILURE_BIN := build/tests/buffer_failure
 FAILURE_SRC := tests/failure/test_buffer_failure.cpp \
 	tests/support/FailingNew.cpp
+FACTORY_FAILURE_BIN := build/tests/factory_failure
+FACTORY_FAILURE_SRC := tests/failure/test_factory_failure.cpp \
+	tests/support/FailingNew.cpp
 NO_ELIDE_BIN := build/tests/unit_no_elide
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
@@ -56,8 +59,13 @@ $(FAILURE_BIN): $(FAILURE_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(FAILURE_SRC) $(NAME) -o $@
 
-failure-test: $(FAILURE_BIN)
+$(FACTORY_FAILURE_BIN): $(FACTORY_FAILURE_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(FACTORY_FAILURE_SRC) $(NAME) -o $@
+
+failure-test: $(FAILURE_BIN) $(FACTORY_FAILURE_BIN)
 	./$(FAILURE_BIN)
+	./$(FACTORY_FAILURE_BIN)
 
 $(NO_ELIDE_BIN): $(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
diff --git a/tests/failure/test_factory_failure.cpp b/tests/failure/test_factory_failure.cpp
new file mode 100644
index 0000000..61e4110
--- /dev/null
+++ b/tests/failure/test_factory_failure.cpp
@@ -0,0 +1,94 @@
+#include "cppf/Factory.hpp"
+#include "support/FailingNew.hpp"
+
+#include <cstring>
+#include <iostream>
+#include <new>
+#include <string>
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
+void testAllocationFailureSweep()
+{
+    const cppf::DefaultFormatterCreator creator;
+    const cppf::UppercaseFormatter seed;
+    const std::string specifications[] = {
+        "prefix=abcdefghijklmnopqrstuvwxyz0123456789",
+        "upper",
+        "suffix=ABCDEFGHIJKLMNOPQRSTUVWXYZ9876543210"};
+    const std::size_t outer_baseline = failing_new::liveBlocks();
+    std::size_t observed = 0;
+    std::size_t index;
+
+    {
+        cppf::FormatPipeline target;
+
+        target.append(seed);
+        failing_new::resetAttempts();
+        cppf::PipelineBuilder::replace(target, creator, specifications, 3);
+        observed = failing_new::attempts();
+        check(target.size() == 3);
+    }
+    check(observed != 0);
+    check(failing_new::liveBlocks() == outer_baseline);
+
+    for (index = 1; index <= observed; ++index)
+    {
+        cppf::FormatPipeline target;
+        bool bad_allocation = false;
+        bool unexpected_exception = false;
+        std::size_t reached_attempt;
+        std::size_t baseline;
+
+        target.append(seed);
+        baseline = failing_new::liveBlocks();
+        failing_new::failOn(index);
+        try
+        {
+            cppf::PipelineBuilder::replace(
+                target, creator, specifications, 3);
+        }
+        catch (const std::bad_alloc &)
+        {
+            bad_allocation = true;
+        }
+        catch (...)
+        {
+            unexpected_exception = true;
+        }
+        failing_new::disableFailure();
+        reached_attempt = failing_new::attempts();
+
+        check(bad_allocation);
+        check(!unexpected_exception);
+        check(reached_attempt == index);
+        check(target.size() == 1);
+        check(std::strcmp(
+                  target.apply(cppf::TextBuffer("keep")).c_str(),
+                  "KEEP") == 0);
+        check(failing_new::liveBlocks() == baseline);
+    }
+    check(failing_new::liveBlocks() == outer_baseline);
+}
+
+}
+
+int main()
+{
+    testAllocationFailureSweep();
+    if (failures != 0)
+        return 1;
+    std::cout << checks << " factory failure checks passed" << std::endl;
+    return 0;
+}
diff --git a/tests/support/TestFormatter.cpp b/tests/support/TestFormatter.cpp
index fabf4ba..a15b68f 100644
--- a/tests/support/TestFormatter.cpp
+++ b/tests/support/TestFormatter.cpp
@@ -5,6 +5,8 @@ namespace test_support
 
 int TestFormatter::live_count_ = 0;
 int TestFormatter::destroyed_count_ = 0;
+std::size_t TestFormatter::clone_attempts_ = 0;
+std::size_t TestFormatter::clone_failure_attempt_ = 0;
 
 TestFormatter::TestFormatter(const cppf::TextBuffer &prefix) : prefix_(prefix)
 {
@@ -25,6 +27,10 @@ TestFormatter::~TestFormatter()
 
 cppf::Formatter *TestFormatter::clone() const
 {
+    ++clone_attempts_;
+    if (clone_failure_attempt_ != 0 &&
+        clone_attempts_ == clone_failure_attempt_)
+        throw CloneFailure();
     return new TestFormatter(*this);
 }
 
@@ -42,6 +48,24 @@ void TestFormatter::resetCounters()
 {
     live_count_ = 0;
     destroyed_count_ = 0;
+    clone_attempts_ = 0;
+    clone_failure_attempt_ = 0;
+}
+
+void TestFormatter::failCloneOn(std::size_t attempt)
+{
+    clone_attempts_ = 0;
+    clone_failure_attempt_ = attempt;
+}
+
+void TestFormatter::disableCloneFailure()
+{
+    clone_failure_attempt_ = 0;
+}
+
+std::size_t TestFormatter::cloneAttempts()
+{
+    return clone_attempts_;
 }
 
 int TestFormatter::liveCount()
diff --git a/tests/support/TestFormatter.hpp b/tests/support/TestFormatter.hpp
index f0dd639..fcf213d 100644
--- a/tests/support/TestFormatter.hpp
+++ b/tests/support/TestFormatter.hpp
@@ -3,9 +3,15 @@
 
 #include "cppf/Formatter.hpp"
 
+#include <cstddef>
+
 namespace test_support
 {
 
+class CloneFailure
+{
+};
+
 class TestFormatter : public cppf::Formatter
 {
 public:
@@ -18,6 +24,9 @@ public:
     virtual const char *name() const;
 
     static void resetCounters();
+    static void failCloneOn(std::size_t attempt);
+    static void disableCloneFailure();
+    static std::size_t cloneAttempts();
     static int liveCount();
     static int destroyedCount();
 
@@ -27,6 +36,8 @@ private:
     cppf::TextBuffer prefix_;
     static int live_count_;
     static int destroyed_count_;
+    static std::size_t clone_attempts_;
+    static std::size_t clone_failure_attempt_;
 };
 
 }
diff --git a/tests/test_factory.cpp b/tests/test_factory.cpp
index 25022ba..2ec8a5c 100644
--- a/tests/test_factory.cpp
+++ b/tests/test_factory.cpp
@@ -1,8 +1,117 @@
 #include "cppf/Factory.hpp"
 #include "support/Test.hpp"
+#include "support/TestFormatter.hpp"
 
 #include <cstring>
 
+namespace
+{
+
+class CreationFailure
+{
+};
+
+class ControlledCreator : public cppf::FormatterCreator
+{
+public:
+    explicit ControlledCreator(std::size_t failure_attempt)
+        : attempts_(0), failure_attempt_(failure_attempt)
+    {
+    }
+
+    virtual cppf::Formatter *create(
+        const std::string &specification) const
+    {
+        ++attempts_;
+        if (failure_attempt_ != 0 && attempts_ == failure_attempt_)
+            throw CreationFailure();
+        return new test_support::TestFormatter(
+            cppf::TextBuffer(specification.c_str()));
+    }
+
+    std::size_t attempts() const
+    {
+        return attempts_;
+    }
+
+private:
+    ControlledCreator(const ControlledCreator &other);
+    ControlledCreator &operator=(const ControlledCreator &other);
+
+    mutable std::size_t attempts_;
+    std::size_t failure_attempt_;
+};
+
+void testCreationFailure(test_support::Suite &suite)
+{
+    const cppf::UppercaseFormatter upper;
+    const std::string specifications[] = {"first", "second", "third"};
+    cppf::FormatPipeline target;
+    const ControlledCreator creator(2);
+    bool threw = false;
+
+    target.append(upper);
+    test_support::TestFormatter::resetCounters();
+    try
+    {
+        cppf::PipelineBuilder::replace(
+            target, creator, specifications, 3);
+    }
+    catch (const CreationFailure &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "pipeline builder preserves creation failure type");
+    suite.check(creator.attempts() == 2,
+                "pipeline builder stops at failed creation");
+    suite.check(test_support::TestFormatter::cloneAttempts() == 1,
+                "creation failure follows one completed clone");
+    suite.check(target.size() == 1,
+                "creation failure preserves target size");
+    suite.check(target.apply(cppf::TextBuffer("keep")) ==
+                    cppf::TextBuffer("KEEP"),
+                "creation failure preserves target behavior");
+    suite.check(test_support::TestFormatter::liveCount() == 0,
+                "creation failure releases candidate formatters");
+}
+
+void testCloneFailure(test_support::Suite &suite)
+{
+    const cppf::UppercaseFormatter upper;
+    const std::string specifications[] = {"first", "second", "third"};
+    cppf::FormatPipeline target;
+    const ControlledCreator creator(0);
+    bool threw = false;
+
+    target.append(upper);
+    test_support::TestFormatter::resetCounters();
+    test_support::TestFormatter::failCloneOn(2);
+    try
+    {
+        cppf::PipelineBuilder::replace(
+            target, creator, specifications, 3);
+    }
+    catch (const test_support::CloneFailure &)
+    {
+        threw = true;
+    }
+    test_support::TestFormatter::disableCloneFailure();
+    suite.check(threw, "pipeline builder preserves clone failure type");
+    suite.check(creator.attempts() == 2,
+                "pipeline builder stops at failed clone");
+    suite.check(test_support::TestFormatter::cloneAttempts() == 2,
+                "pipeline builder reaches configured clone failure");
+    suite.check(target.size() == 1,
+                "clone failure preserves target size");
+    suite.check(target.apply(cppf::TextBuffer("keep")) ==
+                    cppf::TextBuffer("KEEP"),
+                "clone failure preserves target behavior");
+    suite.check(test_support::TestFormatter::liveCount() == 0,
+                "clone failure releases creator and candidate objects");
+}
+
+}
+
 void testFactory(test_support::Suite &suite)
 {
     const cppf::DefaultFormatterCreator creator;
@@ -104,4 +213,7 @@ void testFactory(test_support::Suite &suite)
 
     cppf::PipelineBuilder::replace(pipeline, creator, 0, 0);
     suite.check(pipeline.size() == 0, "pipeline builder accepts empty list");
+
+    testCreationFailure(suite);
+    testCloneFailure(suite);
 }
