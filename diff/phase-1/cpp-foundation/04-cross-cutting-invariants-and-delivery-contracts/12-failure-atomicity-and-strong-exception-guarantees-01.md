# 실패 원자성과 강한 예외 보장

## `feat(buffer): 깊은 복사와 정규 대입 구현`

diff --git a/include/cppf/TextBuffer.hpp b/include/cppf/TextBuffer.hpp
index 7de1a3a..e48df1b 100644
--- a/include/cppf/TextBuffer.hpp
+++ b/include/cppf/TextBuffer.hpp
@@ -11,8 +11,11 @@ class TextBuffer
 public:
     TextBuffer();
     explicit TextBuffer(const char *text);
+    TextBuffer(const TextBuffer &other);
     ~TextBuffer();
 
+    TextBuffer &operator=(const TextBuffer &other);
+
     std::size_t size() const;
     bool empty() const;
     const char *c_str() const;
@@ -21,9 +24,6 @@ public:
     void swap(TextBuffer &other) throw();
 
 private:
-    TextBuffer(const TextBuffer &other);
-    TextBuffer &operator=(const TextBuffer &other);
-
     char *data_;
     std::size_t size_;
 };
diff --git a/src/TextBuffer.cpp b/src/TextBuffer.cpp
index 24399f2..6647d3d 100644
--- a/src/TextBuffer.cpp
+++ b/src/TextBuffer.cpp
@@ -20,11 +20,25 @@ TextBuffer::TextBuffer(const char *text) : data_(0), size_(0)
     std::memcpy(data_, text, size_ + 1);
 }
 
+TextBuffer::TextBuffer(const TextBuffer &other)
+    : data_(new char[other.size_ + 1]), size_(other.size_)
+{
+    std::memcpy(data_, other.data_, size_ + 1);
+}
+
 TextBuffer::~TextBuffer()
 {
     delete[] data_;
 }
 
+TextBuffer &TextBuffer::operator=(const TextBuffer &other)
+{
+    TextBuffer copy(other);
+
+    swap(copy);
+    return *this;
+}
+
 std::size_t TextBuffer::size() const
 {
     return size_;


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


