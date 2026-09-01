# 직접 소유 문자열의 값 의미론

## `feat(buffer): 종료 문자를 포함한 문자열 저장소 소유`

diff --git a/include/cppf/TextBuffer.hpp b/include/cppf/TextBuffer.hpp
new file mode 100644
index 0000000..7de1a3a
--- /dev/null
+++ b/include/cppf/TextBuffer.hpp
@@ -0,0 +1,33 @@
+#ifndef CPPF_TEXT_BUFFER_HPP
+#define CPPF_TEXT_BUFFER_HPP
+
+#include <cstddef>
+
+namespace cppf
+{
+
+class TextBuffer
+{
+public:
+    TextBuffer();
+    explicit TextBuffer(const char *text);
+    ~TextBuffer();
+
+    std::size_t size() const;
+    bool empty() const;
+    const char *c_str() const;
+    char &at(std::size_t index);
+    const char &at(std::size_t index) const;
+    void swap(TextBuffer &other) throw();
+
+private:
+    TextBuffer(const TextBuffer &other);
+    TextBuffer &operator=(const TextBuffer &other);
+
+    char *data_;
+    std::size_t size_;
+};
+
+}
+
+#endif
diff --git a/src/TextBuffer.cpp b/src/TextBuffer.cpp
new file mode 100644
index 0000000..24399f2
--- /dev/null
+++ b/src/TextBuffer.cpp
@@ -0,0 +1,68 @@
+#include "cppf/TextBuffer.hpp"
+
+#include <cstring>
+#include <stdexcept>
+
+namespace cppf
+{
+
+TextBuffer::TextBuffer() : data_(new char[1]), size_(0)
+{
+    data_[0] = '\0';
+}
+
+TextBuffer::TextBuffer(const char *text) : data_(0), size_(0)
+{
+    if (text == 0)
+        text = "";
+    size_ = std::strlen(text);
+    data_ = new char[size_ + 1];
+    std::memcpy(data_, text, size_ + 1);
+}
+
+TextBuffer::~TextBuffer()
+{
+    delete[] data_;
+}
+
+std::size_t TextBuffer::size() const
+{
+    return size_;
+}
+
+bool TextBuffer::empty() const
+{
+    return size_ == 0;
+}
+
+const char *TextBuffer::c_str() const
+{
+    return data_;
+}
+
+char &TextBuffer::at(std::size_t index)
+{
+    if (index >= size_)
+        throw std::out_of_range("text index");
+    return data_[index];
+}
+
+const char &TextBuffer::at(std::size_t index) const
+{
+    if (index >= size_)
+        throw std::out_of_range("text index");
+    return data_[index];
+}
+
+void TextBuffer::swap(TextBuffer &other) throw()
+{
+    char *data = data_;
+    const std::size_t size = size_;
+
+    data_ = other.data_;
+    size_ = other.size_;
+    other.data_ = data;
+    other.size_ = size;
+}
+
+}


## `test(buffer): 저장 크기와 범위 접근 검증`

diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 326d7d0..1661753 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -2,6 +2,7 @@
 
 void testContact(test_support::Suite &suite);
 void testContactBook(test_support::Suite &suite);
+void testTextBuffer(test_support::Suite &suite);
 
 int main()
 {
@@ -9,5 +10,6 @@ int main()
 
     testContact(suite);
     testContactBook(suite);
+    testTextBuffer(suite);
     return suite.result();
 }
diff --git a/tests/test_text_buffer.cpp b/tests/test_text_buffer.cpp
new file mode 100644
index 0000000..58db796
--- /dev/null
+++ b/tests/test_text_buffer.cpp
@@ -0,0 +1,36 @@
+#include "cppf/TextBuffer.hpp"
+#include "support/Test.hpp"
+
+#include <cstring>
+#include <stdexcept>
+
+void testTextBuffer(test_support::Suite &suite)
+{
+    cppf::TextBuffer empty;
+    cppf::TextBuffer value("buffer");
+    cppf::TextBuffer null_value(0);
+    const cppf::TextBuffer &view = value;
+    bool threw = false;
+
+    suite.check(empty.empty(), "text buffer empty state");
+    suite.check(empty.size() == 0, "text buffer empty size");
+    suite.check(std::strcmp(empty.c_str(), "") == 0,
+                "text buffer empty terminator");
+    suite.check(null_value.empty(), "text buffer null input becomes empty");
+    suite.check(value.size() == 6, "text buffer stores length");
+    suite.check(std::strcmp(value.c_str(), "buffer") == 0,
+                "text buffer stores bytes");
+    suite.check(view.at(1) == 'u', "text buffer const access");
+    value.at(0) = 'B';
+    suite.check(std::strcmp(value.c_str(), "Buffer") == 0,
+                "text buffer mutable access");
+    try
+    {
+        value.at(value.size());
+    }
+    catch (const std::out_of_range &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "text buffer checks bounds");
+}


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


## `test(buffer): 복사 독립성과 자기 대입 검증`

diff --git a/tests/test_text_buffer.cpp b/tests/test_text_buffer.cpp
index 58db796..cd21b17 100644
--- a/tests/test_text_buffer.cpp
+++ b/tests/test_text_buffer.cpp
@@ -10,6 +10,10 @@ void testTextBuffer(test_support::Suite &suite)
     cppf::TextBuffer value("buffer");
     cppf::TextBuffer null_value(0);
     const cppf::TextBuffer &view = value;
+    cppf::TextBuffer copy(value);
+    cppf::TextBuffer assigned("old");
+    cppf::TextBuffer chained("chain");
+    const cppf::TextBuffer &self_alias = value;
     bool threw = false;
 
     suite.check(empty.empty(), "text buffer empty state");
@@ -21,6 +25,21 @@ void testTextBuffer(test_support::Suite &suite)
     suite.check(std::strcmp(value.c_str(), "buffer") == 0,
                 "text buffer stores bytes");
     suite.check(view.at(1) == 'u', "text buffer const access");
+    copy.at(0) = 'B';
+    suite.check(std::strcmp(copy.c_str(), "Buffer") == 0,
+                "text buffer copy is mutable");
+    suite.check(std::strcmp(value.c_str(), "buffer") == 0,
+                "text buffer copy owns independent storage");
+    suite.check(&(assigned = value) == &assigned,
+                "text buffer assignment returns self");
+    suite.check(std::strcmp(assigned.c_str(), "buffer") == 0,
+                "text buffer assignment copies bytes");
+    chained = assigned = copy;
+    suite.check(std::strcmp(chained.c_str(), "Buffer") == 0,
+                "text buffer chained assignment");
+    value = self_alias;
+    suite.check(std::strcmp(value.c_str(), "buffer") == 0,
+                "text buffer self assignment preserves value");
     value.at(0) = 'B';
     suite.check(std::strcmp(value.c_str(), "Buffer") == 0,
                 "text buffer mutable access");


## `feat(buffer): 결합·비교·출력 연산 제공`

diff --git a/include/cppf/TextBuffer.hpp b/include/cppf/TextBuffer.hpp
index e48df1b..98e1e6c 100644
--- a/include/cppf/TextBuffer.hpp
+++ b/include/cppf/TextBuffer.hpp
@@ -2,6 +2,7 @@
 #define CPPF_TEXT_BUFFER_HPP
 
 #include <cstddef>
+#include <iosfwd>
 
 namespace cppf
 {
@@ -15,6 +16,7 @@ public:
     ~TextBuffer();
 
     TextBuffer &operator=(const TextBuffer &other);
+    TextBuffer &operator+=(const TextBuffer &other);
 
     std::size_t size() const;
     bool empty() const;
@@ -28,6 +30,12 @@ private:
     std::size_t size_;
 };
 
+TextBuffer operator+(const TextBuffer &left, const TextBuffer &right);
+bool operator==(const TextBuffer &left, const TextBuffer &right);
+bool operator!=(const TextBuffer &left, const TextBuffer &right);
+bool operator<(const TextBuffer &left, const TextBuffer &right);
+std::ostream &operator<<(std::ostream &output, const TextBuffer &value);
+
 }
 
 #endif
diff --git a/src/TextBuffer.cpp b/src/TextBuffer.cpp
index 6647d3d..a96bc87 100644
--- a/src/TextBuffer.cpp
+++ b/src/TextBuffer.cpp
@@ -1,6 +1,8 @@
 #include "cppf/TextBuffer.hpp"
 
 #include <cstring>
+#include <limits>
+#include <ostream>
 #include <stdexcept>
 
 namespace cppf
@@ -39,6 +41,23 @@ TextBuffer &TextBuffer::operator=(const TextBuffer &other)
     return *this;
 }
 
+TextBuffer &TextBuffer::operator+=(const TextBuffer &other)
+{
+    char *joined;
+    std::size_t joined_size;
+
+    if (other.size_ > std::numeric_limits<std::size_t>::max() - size_ - 1)
+        throw std::length_error("text length");
+    joined_size = size_ + other.size_;
+    joined = new char[joined_size + 1];
+    std::memcpy(joined, data_, size_);
+    std::memcpy(joined + size_, other.data_, other.size_ + 1);
+    delete[] data_;
+    data_ = joined;
+    size_ = joined_size;
+    return *this;
+}
+
 std::size_t TextBuffer::size() const
 {
     return size_;
@@ -79,4 +98,32 @@ void TextBuffer::swap(TextBuffer &other) throw()
     other.size_ = size;
 }
 
+TextBuffer operator+(const TextBuffer &left, const TextBuffer &right)
+{
+    TextBuffer result(left);
+
+    result += right;
+    return result;
+}
+
+bool operator==(const TextBuffer &left, const TextBuffer &right)
+{
+    return std::strcmp(left.c_str(), right.c_str()) == 0;
+}
+
+bool operator!=(const TextBuffer &left, const TextBuffer &right)
+{
+    return !(left == right);
+}
+
+bool operator<(const TextBuffer &left, const TextBuffer &right)
+{
+    return std::strcmp(left.c_str(), right.c_str()) < 0;
+}
+
+std::ostream &operator<<(std::ostream &output, const TextBuffer &value)
+{
+    return output << value.c_str();
+}
+
 }


## `feat(buffer): 문자열 결합 CLI 제공`

diff --git a/apps/ex01_text_buffer.cpp b/apps/ex01_text_buffer.cpp
new file mode 100644
index 0000000..9f23ea4
--- /dev/null
+++ b/apps/ex01_text_buffer.cpp
@@ -0,0 +1,18 @@
+#include "cppf/TextBuffer.hpp"
+
+#include <iostream>
+
+int main(int argument_count, char **arguments)
+{
+    if (argument_count != 3)
+    {
+        std::cerr << "usage: ex01_text_buffer LEFT RIGHT" << std::endl;
+        return 1;
+    }
+    const cppf::TextBuffer left(arguments[1]);
+    const cppf::TextBuffer right(arguments[2]);
+    const cppf::TextBuffer joined = left + right;
+
+    std::cout << joined << std::endl;
+    return 0;
+}


## `test(buffer): 연산자와 명령행 결합 결과 검증`

diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index 7fcc8c8..d115730 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -8,3 +8,7 @@ trap 'rm -rf "$temporary_directory"' EXIT HUP INT TERM
 ./bin/ex00_contact_book < tests/fixtures/contact-session.in \
     > "$temporary_directory/contact.out"
 diff -u tests/fixtures/contact-session.out "$temporary_directory/contact.out"
+
+./bin/ex01_text_buffer hello world > "$temporary_directory/text.out"
+printf 'helloworld\n' > "$temporary_directory/text.expected"
+diff -u "$temporary_directory/text.expected" "$temporary_directory/text.out"
diff --git a/tests/test_text_buffer.cpp b/tests/test_text_buffer.cpp
index cd21b17..d262a97 100644
--- a/tests/test_text_buffer.cpp
+++ b/tests/test_text_buffer.cpp
@@ -2,6 +2,7 @@
 #include "support/Test.hpp"
 
 #include <cstring>
+#include <sstream>
 #include <stdexcept>
 
 void testTextBuffer(test_support::Suite &suite)
@@ -14,6 +15,9 @@ void testTextBuffer(test_support::Suite &suite)
     cppf::TextBuffer assigned("old");
     cppf::TextBuffer chained("chain");
     const cppf::TextBuffer &self_alias = value;
+    cppf::TextBuffer left("alpha");
+    const cppf::TextBuffer right("beta");
+    std::ostringstream output;
     bool threw = false;
 
     suite.check(empty.empty(), "text buffer empty state");
@@ -40,6 +44,24 @@ void testTextBuffer(test_support::Suite &suite)
     value = self_alias;
     suite.check(std::strcmp(value.c_str(), "buffer") == 0,
                 "text buffer self assignment preserves value");
+    suite.check(left + right == cppf::TextBuffer("alphabeta"),
+                "text buffer addition composes values");
+    suite.check(left == cppf::TextBuffer("alpha"),
+                "text buffer addition preserves left operand");
+    suite.check(right == cppf::TextBuffer("beta"),
+                "text buffer addition preserves right operand");
+    left += right;
+    suite.check(left == cppf::TextBuffer("alphabeta"),
+                "text buffer compound addition");
+    left += left;
+    suite.check(left == cppf::TextBuffer("alphabetaalphabeta"),
+                "text buffer self concatenation");
+    suite.check(cppf::TextBuffer("a") < cppf::TextBuffer("b"),
+                "text buffer lexical order");
+    suite.check(cppf::TextBuffer("a") != cppf::TextBuffer("b"),
+                "text buffer inequality");
+    output << right;
+    suite.check(output.str() == "beta", "text buffer stream output");
     value.at(0) = 'B';
     suite.check(std::strcmp(value.c_str(), "Buffer") == 0,
                 "text buffer mutable access");


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
