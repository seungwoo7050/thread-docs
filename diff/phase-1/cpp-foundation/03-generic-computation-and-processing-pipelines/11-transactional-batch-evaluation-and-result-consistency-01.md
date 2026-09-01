# 트랜잭션형 일괄 평가와 결과 일관성

## `feat(batch): 작업 결과 값 객체 정의`

diff --git a/include/cppf/BatchEngine.hpp b/include/cppf/BatchEngine.hpp
new file mode 100644
index 0000000..58ff159
--- /dev/null
+++ b/include/cppf/BatchEngine.hpp
@@ -0,0 +1,27 @@
+#ifndef CPPF_BATCH_ENGINE_HPP
+#define CPPF_BATCH_ENGINE_HPP
+
+#include <string>
+
+namespace cppf
+{
+
+class JobResult
+{
+public:
+    JobResult();
+    JobResult(const std::string &name, long value);
+
+    const std::string &name() const;
+    long value() const;
+
+private:
+    std::string name_;
+    long value_;
+};
+
+bool operator==(const JobResult &left, const JobResult &right);
+
+}
+
+#endif
diff --git a/src/BatchEngine.cpp b/src/BatchEngine.cpp
new file mode 100644
index 0000000..42c910a
--- /dev/null
+++ b/src/BatchEngine.cpp
@@ -0,0 +1,30 @@
+#include "cppf/BatchEngine.hpp"
+
+namespace cppf
+{
+
+JobResult::JobResult() : name_(), value_(0)
+{
+}
+
+JobResult::JobResult(const std::string &name, long value)
+    : name_(name), value_(value)
+{
+}
+
+const std::string &JobResult::name() const
+{
+    return name_;
+}
+
+long JobResult::value() const
+{
+    return value_;
+}
+
+bool operator==(const JobResult &left, const JobResult &right)
+{
+    return left.name() == right.name() && left.value() == right.value();
+}
+
+}


## `feat(batch): 입력 문법과 원자 교체 구현`

diff --git a/include/cppf/BatchEngine.hpp b/include/cppf/BatchEngine.hpp
index 58ff159..7ec50e9 100644
--- a/include/cppf/BatchEngine.hpp
+++ b/include/cppf/BatchEngine.hpp
@@ -1,7 +1,9 @@
 #ifndef CPPF_BATCH_ENGINE_HPP
 #define CPPF_BATCH_ENGINE_HPP
 
+#include <iosfwd>
 #include <string>
+#include <vector>
 
 namespace cppf
 {
@@ -22,6 +24,16 @@ private:
 
 bool operator==(const JobResult &left, const JobResult &right);
 
+class BatchEngine
+{
+public:
+    void replace(std::istream &input);
+    const std::vector<JobResult> &results() const;
+
+private:
+    std::vector<JobResult> results_;
+};
+
 }
 
 #endif
diff --git a/src/BatchEngine.cpp b/src/BatchEngine.cpp
index 42c910a..6fb014a 100644
--- a/src/BatchEngine.cpp
+++ b/src/BatchEngine.cpp
@@ -1,5 +1,74 @@
 #include "cppf/BatchEngine.hpp"
 
+#include "cppf/RpnEvaluator.hpp"
+
+#include <istream>
+#include <map>
+#include <stdexcept>
+#include <utility>
+
+namespace
+{
+
+bool isFieldWhitespace(char value)
+{
+    return value == ' ' || value == '\t' || value == '\r' ||
+           value == '\n' || value == '\v' || value == '\f';
+}
+
+std::string trimField(const std::string &field)
+{
+    std::size_t first = 0;
+    std::size_t last = field.size();
+
+    while (first < last && isFieldWhitespace(field[first]))
+        ++first;
+    while (last > first && isFieldWhitespace(field[last - 1]))
+        --last;
+    return field.substr(first, last - first);
+}
+
+bool isNameStart(char value)
+{
+    return (value >= 'A' && value <= 'Z') ||
+           (value >= 'a' && value <= 'z');
+}
+
+bool isNameRest(char value)
+{
+    return isNameStart(value) || (value >= '0' && value <= '9') ||
+           value == '_' || value == '-';
+}
+
+bool isValidName(const std::string &name)
+{
+    if (name.empty() || !isNameStart(name[0]))
+        return false;
+    for (std::size_t index = 1; index < name.size(); ++index)
+    {
+        if (!isNameRest(name[index]))
+            return false;
+    }
+    return true;
+}
+
+void parseLine(const std::string &line,
+               std::string &name,
+               std::string &expression)
+{
+    const std::size_t separator = line.find('|');
+
+    if (separator == std::string::npos ||
+        line.find('|', separator + 1) != std::string::npos)
+        throw std::invalid_argument("invalid batch input");
+    name = trimField(line.substr(0, separator));
+    expression = trimField(line.substr(separator + 1));
+    if (!isValidName(name) || expression.empty())
+        throw std::invalid_argument("invalid batch input");
+}
+
+}
+
 namespace cppf
 {
 
@@ -27,4 +96,33 @@ bool operator==(const JobResult &left, const JobResult &right)
     return left.name() == right.name() && left.value() == right.value();
 }
 
+void BatchEngine::replace(std::istream &input)
+{
+    std::vector<JobResult> candidate;
+    std::map<std::string, long> seen;
+    std::string line;
+
+    while (std::getline(input, line))
+    {
+        std::string name;
+        std::string expression;
+
+        parseLine(line, name, expression);
+        if (seen.find(name) != seen.end())
+            throw std::invalid_argument("invalid batch input");
+        const long value = RpnEvaluator::evaluate(expression);
+
+        seen.insert(std::make_pair(name, value));
+        candidate.push_back(JobResult(name, value));
+    }
+    if (!input.eof() || candidate.empty())
+        throw std::invalid_argument("invalid batch input");
+    results_.swap(candidate);
+}
+
+const std::vector<JobResult> &BatchEngine::results() const
+{
+    return results_;
+}
+
 }


## `feat(batch): 결과 정렬과 직렬화 제공`

diff --git a/include/cppf/BatchEngine.hpp b/include/cppf/BatchEngine.hpp
index 7ec50e9..ee914cf 100644
--- a/include/cppf/BatchEngine.hpp
+++ b/include/cppf/BatchEngine.hpp
@@ -29,6 +29,7 @@ class BatchEngine
 public:
     void replace(std::istream &input);
     const std::vector<JobResult> &results() const;
+    void write(std::ostream &output) const;
 
 private:
     std::vector<JobResult> results_;
diff --git a/src/BatchEngine.cpp b/src/BatchEngine.cpp
index 6fb014a..4ffed99 100644
--- a/src/BatchEngine.cpp
+++ b/src/BatchEngine.cpp
@@ -2,8 +2,12 @@
 
 #include "cppf/RpnEvaluator.hpp"
 
+#include <algorithm>
 #include <istream>
+#include <locale>
 #include <map>
+#include <ostream>
+#include <sstream>
 #include <stdexcept>
 #include <utility>
 
@@ -42,9 +46,11 @@ bool isNameRest(char value)
 
 bool isValidName(const std::string &name)
 {
+    std::size_t index;
+
     if (name.empty() || !isNameStart(name[0]))
         return false;
-    for (std::size_t index = 1; index < name.size(); ++index)
+    for (index = 1; index < name.size(); ++index)
     {
         if (!isNameRest(name[index]))
             return false;
@@ -52,6 +58,14 @@ bool isValidName(const std::string &name)
     return true;
 }
 
+bool resultLess(const cppf::JobResult &left,
+                const cppf::JobResult &right)
+{
+    if (left.value() != right.value())
+        return left.value() < right.value();
+    return left.name() < right.name();
+}
+
 void parseLine(const std::string &line,
                std::string &name,
                std::string &expression)
@@ -117,6 +131,7 @@ void BatchEngine::replace(std::istream &input)
     }
     if (!input.eof() || candidate.empty())
         throw std::invalid_argument("invalid batch input");
+    std::sort(candidate.begin(), candidate.end(), resultLess);
     results_.swap(candidate);
 }
 
@@ -125,4 +140,18 @@ const std::vector<JobResult> &BatchEngine::results() const
     return results_;
 }
 
+void BatchEngine::write(std::ostream &output) const
+{
+    std::ostringstream rendered;
+    std::size_t index;
+
+    rendered.imbue(std::locale::classic());
+    for (index = 0; index < results_.size(); ++index)
+        rendered << results_[index].value() << " | "
+                 << results_[index].name() << '\n';
+    const std::string text = rendered.str();
+
+    output.write(text.data(), static_cast<std::streamsize>(text.size()));
+}
+
 }


## `feat(batch): batch engine CLI 제공`

diff --git a/apps/ex05_batch_engine.cpp b/apps/ex05_batch_engine.cpp
new file mode 100644
index 0000000..6a22355
--- /dev/null
+++ b/apps/ex05_batch_engine.cpp
@@ -0,0 +1,36 @@
+#include "cppf/BatchEngine.hpp"
+#include "cppf/RpnEvaluator.hpp"
+
+#include <exception>
+#include <iostream>
+#include <string>
+
+int main(int argument_count, char **arguments)
+{
+    try
+    {
+        if (argument_count == 3 && std::string(arguments[1]) == "rpn")
+        {
+            const long value = cppf::RpnEvaluator::evaluate(arguments[2]);
+
+            std::cout << value << std::endl;
+            return 0;
+        }
+        if (argument_count == 2 && std::string(arguments[1]) == "batch")
+        {
+            cppf::BatchEngine engine;
+
+            engine.replace(std::cin);
+            engine.write(std::cout);
+            return 0;
+        }
+    }
+    catch (const std::exception &error)
+    {
+        std::cerr << error.what() << std::endl;
+        return 1;
+    }
+    std::cerr << "usage: ex05_batch_engine rpn EXPRESSION | batch"
+              << std::endl;
+    return 1;
+}


## `test(batch): 입력 검증·정렬·CLI 결과 검증`

diff --git a/Makefile b/Makefile
index 389fb9c..aaf2baf 100644
--- a/Makefile
+++ b/Makefile
@@ -90,6 +90,8 @@ test-contract:
 		tests/compile/template_headers.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/rpn_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/batch_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index 6069752..d1fe480 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -100,3 +100,25 @@ fi
 test ! -s "$temporary_directory/address-overflow.out"
 diff -u "$temporary_directory/address-failure.expected" \
     "$temporary_directory/address-overflow.err"
+
+./bin/ex05_batch_engine rpn '8 3 -' \
+    > "$temporary_directory/rpn.out"
+printf '5\n' > "$temporary_directory/rpn.expected"
+diff -u "$temporary_directory/rpn.expected" \
+    "$temporary_directory/rpn.out"
+
+./bin/ex05_batch_engine batch < tests/fixtures/batch-basic.in \
+    > "$temporary_directory/batch.out"
+diff -u tests/fixtures/batch-basic.out "$temporary_directory/batch.out"
+
+if ./bin/ex05_batch_engine batch < tests/fixtures/batch-duplicate.in \
+    > "$temporary_directory/batch-failure.out" \
+    2> "$temporary_directory/batch-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/batch-failure.out"
+printf 'invalid batch input\n' \
+    > "$temporary_directory/batch-failure.expected"
+diff -u "$temporary_directory/batch-failure.expected" \
+    "$temporary_directory/batch-failure.err"
diff --git a/tests/compile/batch_headers.cpp b/tests/compile/batch_headers.cpp
new file mode 100644
index 0000000..5631abf
--- /dev/null
+++ b/tests/compile/batch_headers.cpp
@@ -0,0 +1,13 @@
+#include "cppf/BatchEngine.hpp"
+#include "cppf/BatchEngine.hpp"
+
+#include <sstream>
+
+int main()
+{
+    std::istringstream input("alpha | 2 3 +\n");
+    cppf::BatchEngine engine;
+
+    engine.replace(input);
+    return engine.results().at(0).value() != 5;
+}
diff --git a/tests/fixtures/batch-basic.in b/tests/fixtures/batch-basic.in
new file mode 100644
index 0000000..0c46d35
--- /dev/null
+++ b/tests/fixtures/batch-basic.in
@@ -0,0 +1,3 @@
+zeta | 2 3 +
+alpha | 10 5 -
+beta | 3 4 +
diff --git a/tests/fixtures/batch-basic.out b/tests/fixtures/batch-basic.out
new file mode 100644
index 0000000..7c0789d
--- /dev/null
+++ b/tests/fixtures/batch-basic.out
@@ -0,0 +1,3 @@
+5 | alpha
+5 | zeta
+7 | beta
diff --git a/tests/fixtures/batch-duplicate.in b/tests/fixtures/batch-duplicate.in
new file mode 100644
index 0000000..1f86922
--- /dev/null
+++ b/tests/fixtures/batch-duplicate.in
@@ -0,0 +1,2 @@
+alpha | 2 3 +
+alpha | 8 1 -
diff --git a/tests/test_batch_engine.cpp b/tests/test_batch_engine.cpp
new file mode 100644
index 0000000..aed2583
--- /dev/null
+++ b/tests/test_batch_engine.cpp
@@ -0,0 +1,137 @@
+#include "cppf/BatchEngine.hpp"
+#include "support/Test.hpp"
+
+#include <ios>
+#include <locale>
+#include <sstream>
+#include <stdexcept>
+#include <string>
+
+namespace
+{
+
+class CommaPunctuation : public std::numpunct<char>
+{
+protected:
+    virtual char do_decimal_point() const
+    {
+        return ',';
+    }
+};
+
+std::string writeBatch(const cppf::BatchEngine &engine)
+{
+    std::ostringstream output;
+
+    engine.write(output);
+    return output.str();
+}
+
+}
+
+void testBatchEngine(test_support::Suite &suite)
+{
+    const cppf::JobResult empty;
+    const cppf::JobResult alpha("alpha", 5);
+    const cppf::JobResult alpha_copy("alpha", 5);
+    const cppf::JobResult beta("beta", 5);
+
+    suite.check(empty.name().empty() && empty.value() == 0,
+                "job result has empty default value");
+    suite.check(alpha.name() == "alpha" && alpha.value() == 5,
+                "job result preserves name and value");
+    suite.check(alpha == alpha_copy && !(alpha == beta),
+                "job result equality compares both fields");
+
+    cppf::BatchEngine engine;
+    suite.check(engine.results().empty() && writeBatch(engine).empty(),
+                "batch engine starts empty");
+
+    {
+        std::istringstream input(
+            "zeta | 2 3 +\n"
+            "alpha | 10 5 -\n"
+            "beta | 3 4 +");
+
+        engine.replace(input);
+    }
+    suite.check(engine.results().size() == 3,
+                "batch engine owns parsed jobs after input lifetime");
+    suite.check(engine.results()[0] == cppf::JobResult("alpha", 5) &&
+                    engine.results()[1] == cppf::JobResult("zeta", 5) &&
+                    engine.results()[2] == cppf::JobResult("beta", 7),
+                "batch engine sorts by value then name");
+    const std::string original =
+        "5 | alpha\n5 | zeta\n7 | beta\n";
+    suite.check(writeBatch(engine) == original,
+                "batch engine writes deterministic decimal rows");
+
+    const cppf::JobResult *original_first = &engine.results()[0];
+    bool duplicate = false;
+    try
+    {
+        std::istringstream input(
+            "other | 1\nother | 2\n");
+        engine.replace(input);
+    }
+    catch (const std::invalid_argument &error)
+    {
+        duplicate =
+            std::string(error.what()) == "invalid batch input";
+    }
+    suite.check(duplicate && &engine.results()[0] == original_first &&
+                    writeBatch(engine) == original,
+                "duplicate name preserves prior batch and references");
+
+    bool invalid_rpn = false;
+    try
+    {
+        std::istringstream input(
+            "other | 1\nsecond | 1 +\n");
+        engine.replace(input);
+    }
+    catch (const std::invalid_argument &error)
+    {
+        invalid_rpn =
+            std::string(error.what()) == "invalid rpn expression";
+    }
+    suite.check(invalid_rpn && &engine.results()[0] == original_first &&
+                    writeBatch(engine) == original,
+                "invalid rpn preserves prior batch and references");
+
+    bool empty_input = false;
+    try
+    {
+        std::istringstream input("");
+        engine.replace(input);
+    }
+    catch (const std::invalid_argument &error)
+    {
+        empty_input =
+            std::string(error.what()) == "invalid batch input";
+    }
+    suite.check(empty_input && writeBatch(engine) == original,
+                "empty input preserves prior batch");
+
+    std::ostringstream configured;
+    configured.setf(std::ios::hex, std::ios::basefield);
+    configured.setf(std::ios::showpos);
+    configured.setf(std::ios::left, std::ios::adjustfield);
+    configured.fill('#');
+    configured.width(80);
+    configured.precision(2);
+    configured.imbue(std::locale(std::locale::classic(),
+                                new CommaPunctuation));
+    const std::ios::fmtflags flags = configured.flags();
+    const char fill = configured.fill();
+    const std::streamsize width = configured.width();
+    const std::streamsize precision = configured.precision();
+
+    engine.write(configured);
+    suite.check(configured.str() == original,
+                "batch output ignores caller formatting and locale");
+    suite.check(configured.flags() == flags && configured.fill() == fill &&
+                    configured.width() == width &&
+                    configured.precision() == precision,
+                "batch output preserves caller formatting state");
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index e7ca77d..68ecfba 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -12,6 +12,7 @@ void testRuntimeType(test_support::Suite &suite);
 void testSerializer(test_support::Suite &suite);
 void testRandomAccessBatch(test_support::Suite &suite);
 void testRpnEvaluator(test_support::Suite &suite);
+void testBatchEngine(test_support::Suite &suite);
 
 int main()
 {
@@ -29,5 +30,6 @@ int main()
     testSerializer(suite);
     testRandomAccessBatch(suite);
     testRpnEvaluator(suite);
+    testBatchEngine(suite);
     return suite.result();
 }


## `feat(batch): 두 container의 정렬 결과 대조`

diff --git a/src/BatchEngine.cpp b/src/BatchEngine.cpp
index 4ffed99..a576739 100644
--- a/src/BatchEngine.cpp
+++ b/src/BatchEngine.cpp
@@ -1,8 +1,9 @@
 #include "cppf/BatchEngine.hpp"
 
+#include "cppf/RandomAccessBatch.hpp"
 #include "cppf/RpnEvaluator.hpp"
 
-#include <algorithm>
+#include <deque>
 #include <istream>
 #include <locale>
 #include <map>
@@ -112,7 +113,8 @@ bool operator==(const JobResult &left, const JobResult &right)
 
 void BatchEngine::replace(std::istream &input)
 {
-    std::vector<JobResult> candidate;
+    RandomAccessBatch<JobResult> vector_batch;
+    RandomAccessBatch<JobResult, std::deque<JobResult> > deque_batch;
     std::map<std::string, long> seen;
     std::string line;
 
@@ -125,13 +127,22 @@ void BatchEngine::replace(std::istream &input)
         if (seen.find(name) != seen.end())
             throw std::invalid_argument("invalid batch input");
         const long value = RpnEvaluator::evaluate(expression);
+        const JobResult result(name, value);
 
         seen.insert(std::make_pair(name, value));
-        candidate.push_back(JobResult(name, value));
+        vector_batch.push_back(result);
+        deque_batch.push_back(result);
     }
-    if (!input.eof() || candidate.empty())
+    if (!input.eof() || vector_batch.empty())
         throw std::invalid_argument("invalid batch input");
-    std::sort(candidate.begin(), candidate.end(), resultLess);
+    vector_batch.sort(resultLess);
+    deque_batch.sort(resultLess);
+    if (!equal_ranges(vector_batch.begin(), vector_batch.end(),
+                      deque_batch.begin(), deque_batch.end()))
+        throw std::logic_error("batch container disagreement");
+    std::vector<JobResult> candidate(
+        vector_batch.begin(), vector_batch.end());
+
     results_.swap(candidate);
 }
 


## `test(batch): 입력 순열과 출력 결정성 검증`

diff --git a/tests/test_batch_engine.cpp b/tests/test_batch_engine.cpp
index aed2583..5cce7e7 100644
--- a/tests/test_batch_engine.cpp
+++ b/tests/test_batch_engine.cpp
@@ -27,6 +27,15 @@ std::string writeBatch(const cppf::BatchEngine &engine)
     return output.str();
 }
 
+std::string runBatch(const std::string &text)
+{
+    std::istringstream input(text);
+    cppf::BatchEngine engine;
+
+    engine.replace(input);
+    return writeBatch(engine);
+}
+
 }
 
 void testBatchEngine(test_support::Suite &suite)
@@ -134,4 +143,39 @@ void testBatchEngine(test_support::Suite &suite)
                     configured.width() == width &&
                     configured.precision() == precision,
                 "batch output preserves caller formatting state");
+
+    const std::string ordered =
+        "-2 | neg\n"
+        "0 | zero\n"
+        "2 | pos\n"
+        "5 | Alpha\n"
+        "5 | alpha\n"
+        "5 | beta\n"
+        "5 | zeta\n";
+    const std::string first_permutation =
+        "zeta | 5\n"
+        "neg | -2\n"
+        "alpha | 5\n"
+        "pos | 2\n"
+        "beta | 5\n"
+        "zero | 0\n"
+        "Alpha | 5\n";
+    const std::string second_permutation =
+        "Alpha | 5\n"
+        "zero | 0\n"
+        "beta | 5\n"
+        "pos | 2\n"
+        "alpha | 5\n"
+        "neg | -2\n"
+        "zeta | 5";
+
+    suite.check(runBatch(first_permutation) == ordered,
+                "batch orders vector and deque results identically");
+    suite.check(runBatch(second_permutation) == ordered,
+                "batch output is invariant under input permutation");
+    suite.check(runBatch("only | 7") == "7 | only\n",
+                "batch container comparison handles one element");
+    suite.check(writeBatch(engine) == original &&
+                    writeBatch(engine) == original,
+                "batch repeated output is byte-identical");
 }


