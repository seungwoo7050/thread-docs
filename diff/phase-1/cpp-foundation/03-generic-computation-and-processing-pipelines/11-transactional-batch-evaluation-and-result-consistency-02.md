## `fix(batch): 입력 stream 종료 상태를 명확히 구분`

diff --git a/src/BatchEngine.cpp b/src/BatchEngine.cpp
index a576739..c1d9cc7 100644
--- a/src/BatchEngine.cpp
+++ b/src/BatchEngine.cpp
@@ -82,6 +82,22 @@ void parseLine(const std::string &line,
         throw std::invalid_argument("invalid batch input");
 }
 
+bool readLine(std::istream &input, std::string &line)
+{
+    char value;
+
+    line.clear();
+    while (input.get(value))
+    {
+        if (value == '\n')
+            return true;
+        line.push_back(value);
+    }
+    if (!input.eof())
+        throw std::invalid_argument("invalid batch input");
+    return !line.empty();
+}
+
 }
 
 namespace cppf
@@ -118,7 +134,7 @@ void BatchEngine::replace(std::istream &input)
     std::map<std::string, long> seen;
     std::string line;
 
-    while (std::getline(input, line))
+    while (readLine(input, line))
     {
         std::string name;
         std::string expression;
@@ -133,7 +149,7 @@ void BatchEngine::replace(std::istream &input)
         vector_batch.push_back(result);
         deque_batch.push_back(result);
     }
-    if (!input.eof() || vector_batch.empty())
+    if (vector_batch.empty())
         throw std::invalid_argument("invalid batch input");
     vector_batch.sort(resultLess);
     deque_batch.sort(resultLess);


## `test(batch): 입력·산술·할당 실패 뒤 상태 복원 검증`

diff --git a/Makefile b/Makefile
index aaf2baf..b7dd496 100644
--- a/Makefile
+++ b/Makefile
@@ -28,6 +28,9 @@ FAILURE_SRC := tests/failure/test_buffer_failure.cpp \
 FACTORY_FAILURE_BIN := build/tests/factory_failure
 FACTORY_FAILURE_SRC := tests/failure/test_factory_failure.cpp \
 	tests/support/FailingNew.cpp
+BATCH_FAILURE_BIN := build/tests/batch_failure
+BATCH_FAILURE_SRC := tests/failure/test_batch_failure.cpp \
+	tests/support/FailingNew.cpp
 NO_ELIDE_BIN := build/tests/unit_no_elide
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
@@ -63,9 +66,14 @@ $(FACTORY_FAILURE_BIN): $(FACTORY_FAILURE_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(FACTORY_FAILURE_SRC) $(NAME) -o $@
 
-failure-test: $(FAILURE_BIN) $(FACTORY_FAILURE_BIN)
+$(BATCH_FAILURE_BIN): $(BATCH_FAILURE_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(BATCH_FAILURE_SRC) $(NAME) -o $@
+
+failure-test: $(FAILURE_BIN) $(FACTORY_FAILURE_BIN) $(BATCH_FAILURE_BIN)
 	./$(FAILURE_BIN)
 	./$(FACTORY_FAILURE_BIN)
+	./$(BATCH_FAILURE_BIN)
 
 $(NO_ELIDE_BIN): $(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
@@ -104,6 +112,10 @@ test-contract:
 		tests/compile/serializer_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/serializer_const_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/rpn_evaluator_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/batch_results_mutation_fail.cpp >/dev/null 2>&1
 
 test-integration: $(APP_BIN)
 	sh tests/check_cli.sh
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index d1fe480..c58de77 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -122,3 +122,15 @@ printf 'invalid batch input\n' \
     > "$temporary_directory/batch-failure.expected"
 diff -u "$temporary_directory/batch-failure.expected" \
     "$temporary_directory/batch-failure.err"
+
+if ./bin/ex05_batch_engine batch < tests/fixtures/batch-invalid-rpn.in \
+    > "$temporary_directory/batch-rpn-failure.out" \
+    2> "$temporary_directory/batch-rpn-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/batch-rpn-failure.out"
+printf 'invalid rpn expression\n' \
+    > "$temporary_directory/batch-rpn-failure.expected"
+diff -u "$temporary_directory/batch-rpn-failure.expected" \
+    "$temporary_directory/batch-rpn-failure.err"
diff --git a/tests/compile/batch_results_mutation_fail.cpp b/tests/compile/batch_results_mutation_fail.cpp
new file mode 100644
index 0000000..af1ac9d
--- /dev/null
+++ b/tests/compile/batch_results_mutation_fail.cpp
@@ -0,0 +1,8 @@
+#include "cppf/BatchEngine.hpp"
+
+int main()
+{
+    cppf::BatchEngine engine;
+    engine.results().push_back(cppf::JobResult("value", 1));
+    return 0;
+}
diff --git a/tests/compile/rpn_evaluator_private_fail.cpp b/tests/compile/rpn_evaluator_private_fail.cpp
new file mode 100644
index 0000000..e4fdd83
--- /dev/null
+++ b/tests/compile/rpn_evaluator_private_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/RpnEvaluator.hpp"
+
+int main()
+{
+    cppf::RpnEvaluator evaluator;
+    return 0;
+}
diff --git a/tests/failure/test_batch_failure.cpp b/tests/failure/test_batch_failure.cpp
new file mode 100644
index 0000000..fc6b5a7
--- /dev/null
+++ b/tests/failure/test_batch_failure.cpp
@@ -0,0 +1,118 @@
+#include "cppf/BatchEngine.hpp"
+#include "support/FailingNew.hpp"
+
+#include <iostream>
+#include <new>
+#include <sstream>
+#include <string>
+
+namespace
+{
+
+unsigned int checks = 0;
+unsigned int failures = 0;
+unsigned int first_failure = 0;
+
+void check(bool condition)
+{
+    ++checks;
+    if (!condition)
+    {
+        if (first_failure == 0)
+            first_failure = checks;
+        ++failures;
+    }
+}
+
+void seed(cppf::BatchEngine &engine, const std::string &text)
+{
+    std::istringstream input(text);
+
+    engine.replace(input);
+}
+
+void testAllocationFailureSweep()
+{
+    const std::string seed_text = "seed | 7";
+    const std::string replacement_text =
+        "long_alpha_name_0123456789 | 10 20 +\n"
+        "long_beta_name_abcdefghijklmnopqrstuvwxyz | 50 8 -\n"
+        "long_gamma_name_ABCDEFGHIJKLMNOPQRSTUVWXYZ | 6 7 *";
+    const std::string expected_seed = "7 | seed\n";
+    const std::size_t outer_baseline = failing_new::liveBlocks();
+    std::size_t observed = 0;
+    std::size_t index;
+
+    {
+        std::istringstream seed_input(seed_text);
+        std::istringstream replacement_input(replacement_text);
+        cppf::BatchEngine engine;
+
+        engine.replace(seed_input);
+        failing_new::resetAttempts();
+        engine.replace(replacement_input);
+        observed = failing_new::attempts();
+        check(engine.results().size() == 3);
+    }
+    check(observed != 0);
+    check(failing_new::liveBlocks() == outer_baseline);
+
+    for (index = 1; index <= observed; ++index)
+    {
+        {
+            std::istringstream replacement_input(replacement_text);
+            cppf::BatchEngine engine;
+            bool bad_allocation = false;
+            bool unexpected_exception = false;
+            std::size_t reached_attempt;
+            std::size_t baseline;
+
+            seed(engine, seed_text);
+            baseline = failing_new::liveBlocks();
+            failing_new::failOn(index);
+            try
+            {
+                engine.replace(replacement_input);
+            }
+            catch (const std::bad_alloc &)
+            {
+                bad_allocation = true;
+            }
+            catch (...)
+            {
+                unexpected_exception = true;
+            }
+            failing_new::disableFailure();
+            reached_attempt = failing_new::attempts();
+
+            check(bad_allocation);
+            check(!unexpected_exception);
+            check(reached_attempt == index);
+            check(engine.results().size() == 1);
+            check(engine.results()[0] == cppf::JobResult("seed", 7));
+            {
+                std::ostringstream output;
+
+                engine.write(output);
+                check(output.str() == expected_seed);
+            }
+            check(failing_new::liveBlocks() == baseline);
+        }
+        check(failing_new::liveBlocks() == outer_baseline);
+    }
+}
+
+}
+
+int main()
+{
+    testAllocationFailureSweep();
+    if (failures != 0)
+    {
+        std::cerr << failures << " batch failure checks failed; first: "
+                  << first_failure << std::endl;
+        return 1;
+    }
+    std::cout << checks << " batch failure checks passed" << std::endl;
+    return 0;
+}
diff --git a/tests/fixtures/batch-invalid-rpn.in b/tests/fixtures/batch-invalid-rpn.in
new file mode 100644
index 0000000..c919d16
--- /dev/null
+++ b/tests/fixtures/batch-invalid-rpn.in
@@ -0,0 +1 @@
+alpha | 1 +
diff --git a/tests/test_batch_engine.cpp b/tests/test_batch_engine.cpp
index 5cce7e7..5c3a430 100644
--- a/tests/test_batch_engine.cpp
+++ b/tests/test_batch_engine.cpp
@@ -2,6 +2,7 @@
 #include "support/Test.hpp"
 
 #include <ios>
+#include <limits>
 #include <locale>
 #include <sstream>
 #include <stdexcept>
@@ -36,6 +37,61 @@ std::string runBatch(const std::string &text)
     return writeBatch(engine);
 }
 
+std::string decimalLong(long value)
+{
+    std::ostringstream output;
+
+    output << value;
+    return output.str();
+}
+
+void checkInvalidPreserves(test_support::Suite &suite,
+                           cppf::BatchEngine &engine,
+                           const std::string &text,
+                           const char *expected_error,
+                           const char *label)
+{
+    const std::string before = writeBatch(engine);
+    const cppf::JobResult *first = &engine.results()[0];
+    bool invalid = false;
+
+    try
+    {
+        std::istringstream input(text);
+        engine.replace(input);
+    }
+    catch (const std::invalid_argument &error)
+    {
+        invalid = std::string(error.what()) == expected_error;
+    }
+    suite.check(invalid && &engine.results()[0] == first &&
+                    writeBatch(engine) == before,
+                label);
+}
+
+void checkOverflowPreserves(test_support::Suite &suite,
+                            cppf::BatchEngine &engine,
+                            const std::string &text,
+                            const char *label)
+{
+    const std::string before = writeBatch(engine);
+    const cppf::JobResult *first = &engine.results()[0];
+    bool overflow = false;
+
+    try
+    {
+        std::istringstream input(text);
+        engine.replace(input);
+    }
+    catch (const std::overflow_error &error)
+    {
+        overflow = std::string(error.what()) == "rpn overflow";
+    }
+    suite.check(overflow && &engine.results()[0] == first &&
+                    writeBatch(engine) == before,
+                label);
+}
+
 }
 
 void testBatchEngine(test_support::Suite &suite)
@@ -178,4 +234,70 @@ void testBatchEngine(test_support::Suite &suite)
     suite.check(writeBatch(engine) == original &&
                     writeBatch(engine) == original,
                 "batch repeated output is byte-identical");
+
+    suite.check(runBatch("\talpha\t | \t1 2 +\t\r\nA9_- | 4") ==
+                    "3 | alpha\n4 | A9_-\n",
+                "batch trims fields and accepts crlf without final newline");
+    checkInvalidPreserves(suite, engine, "   ", "invalid batch input",
+                          "batch rejects spaces-only line transactionally");
+    checkInvalidPreserves(suite, engine, "one | 1\n\ntwo | 2",
+                          "invalid batch input",
+                          "batch rejects blank middle line transactionally");
+    checkInvalidPreserves(suite, engine, "one | 1\n\n",
+                          "invalid batch input",
+                          "batch rejects trailing blank line transactionally");
+    checkInvalidPreserves(suite, engine, "one 1", "invalid batch input",
+                          "batch rejects missing separator transactionally");
+    checkInvalidPreserves(suite, engine, "one | 1 | 2",
+                          "invalid batch input",
+                          "batch rejects extra separator transactionally");
+    checkInvalidPreserves(suite, engine, " | 1", "invalid batch input",
+                          "batch rejects empty name transactionally");
+    checkInvalidPreserves(suite, engine, "one | ", "invalid batch input",
+                          "batch rejects empty expression transactionally");
+    checkInvalidPreserves(suite, engine, "9one | 1",
+                          "invalid batch input",
+                          "batch rejects digit-first name transactionally");
+    checkInvalidPreserves(suite, engine, "one two | 1",
+                          "invalid batch input",
+                          "batch rejects name whitespace transactionally");
+    checkInvalidPreserves(suite, engine, "one.two | 1",
+                          "invalid batch input",
+                          "batch rejects name punctuation transactionally");
+    checkInvalidPreserves(suite, engine,
+                          std::string("a\0b", 3) + " | 1",
+                          "invalid batch input",
+                          "batch rejects embedded nul name transactionally");
+    checkInvalidPreserves(suite, engine,
+                          std::string(1, static_cast<char>(0x80)) + " | 1",
+                          "invalid batch input",
+                          "batch rejects non-ascii name transactionally");
+    checkInvalidPreserves(suite, engine,
+                          "same | 1\nsame | 1",
+                          "invalid batch input",
+                          "batch rejects exact duplicate transactionally");
+    checkInvalidPreserves(suite, engine, "zero | 1 0 /",
+                          "invalid rpn expression",
+                          "batch propagates division failure transactionally");
+    checkOverflowPreserves(
+        suite, engine,
+        "large | " + decimalLong(std::numeric_limits<long>::max()) +
+            " 1 +",
+        "batch propagates rpn overflow transactionally");
+
+    std::istringstream broken("other | 1");
+    broken.setstate(std::ios::badbit);
+    const cppf::JobResult *broken_first = &engine.results()[0];
+    bool bad_stream = false;
+    try
+    {
+        engine.replace(broken);
+    }
+    catch (const std::invalid_argument &error)
+    {
+        bad_stream = std::string(error.what()) == "invalid batch input";
+    }
+    suite.check(bad_stream && &engine.results()[0] == broken_first &&
+                    writeBatch(engine) == original,
+                "batch bad stream preserves prior result and reference");
 }


