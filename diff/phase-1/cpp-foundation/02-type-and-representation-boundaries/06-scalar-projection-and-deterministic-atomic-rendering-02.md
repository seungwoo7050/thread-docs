## `test(boundary): 변환·배치 속성과 대용량 경계 검증`

diff --git a/Makefile b/Makefile
index 1bf8937..4464307 100644
--- a/Makefile
+++ b/Makefile
@@ -41,6 +41,8 @@ CONTACT_FAILURE_SRC := tests/failure/test_contact_failure.cpp \
 NO_ELIDE_BIN := build/tests/unit_no_elide
 PUBLIC_CONTRACT_BIN := build/tests/public_contract
 PUBLIC_CONTRACT_SRC := tests/integration/test_public_contract.cpp
+PROPERTY_BIN := build/tests/boundary_properties
+PROPERTY_SRC := tests/property/test_boundary_properties.cpp
 SANITIZER_BIN := build/tests/unit_sanitize
 SANITIZER_FLAGS := -O1 -fsanitize=undefined -fno-sanitize-recover=all \
 	-fno-omit-frame-pointer -g
@@ -48,7 +50,8 @@ RELEASE_BIN := $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
 	test-integration test-consumer test-sanitize test-leak check-archive \
-	check-dependencies check-determinism test check clean fclean re
+	check-dependencies check-determinism test-property test check clean \
+	fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -182,6 +185,13 @@ test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN) test-consumer
 	sh tests/check_cli.sh
 	./$(PUBLIC_CONTRACT_BIN)
 
+$(PROPERTY_BIN): $(PROPERTY_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(PROPERTY_SRC) $(NAME) -o $@
+
+test-property: $(PROPERTY_BIN)
+	sh tests/run_with_timeout.sh 30 ./$(PROPERTY_BIN)
+
 $(SANITIZER_BIN): $(SRC) $(TEST_SRC) $(TEST_SUPPORT_SRC)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(SANITIZER_FLAGS) \
@@ -205,7 +215,8 @@ check-determinism: $(APP_BIN)
 	LC_ALL=C LANG=C TZ=UTC sh tests/check_cli.sh
 	LC_ALL=C LANG=C TZ=UTC sh tests/check_cli.sh
 
-test: test-unit failure-test test-no-elide test-contract test-integration
+test: test-unit failure-test test-no-elide test-contract test-integration \
+	test-property
 
 check:
 	git diff --check
diff --git a/tests/property/test_boundary_properties.cpp b/tests/property/test_boundary_properties.cpp
new file mode 100644
index 0000000..036aeb3
--- /dev/null
+++ b/tests/property/test_boundary_properties.cpp
@@ -0,0 +1,286 @@
+#include "cppf/BatchEngine.hpp"
+#include "cppf/RpnEvaluator.hpp"
+#include "cppf/ScalarConverter.hpp"
+
+#include <algorithm>
+#include <iomanip>
+#include <iostream>
+#include <locale>
+#include <sstream>
+#include <stdexcept>
+#include <string>
+#include <vector>
+
+namespace
+{
+
+const unsigned long fixed_seed = 0x13579BDFUL;
+unsigned int checks = 0;
+unsigned int failures = 0;
+unsigned long random_state = fixed_seed;
+std::string first_failure;
+
+void check(bool condition, const std::string &context)
+{
+    ++checks;
+    if (!condition)
+    {
+        if (first_failure.empty())
+            first_failure = context;
+        ++failures;
+    }
+}
+
+const char *booleanText(bool value)
+{
+    return value ? "true" : "false";
+}
+
+unsigned long nextRandom()
+{
+    random_state =
+        (random_state * 1103515245UL + 12345UL) & 0x7FFFFFFFUL;
+    return random_state;
+}
+
+std::string decimal(long value)
+{
+    std::ostringstream output;
+
+    output.imbue(std::locale::classic());
+    output << value;
+    return output.str();
+}
+
+std::string escaped(const std::string &text)
+{
+    std::string result;
+    std::size_t index;
+
+    for (index = 0; index < text.size(); ++index)
+    {
+        if (text[index] == '\n')
+            result += "\\n";
+        else if (text[index] == '\r')
+            result += "\\r";
+        else if (text[index] == '\t')
+            result += "\\t";
+        else
+            result += text[index];
+    }
+    return result;
+}
+
+std::string scalarOutput(const std::string &literal)
+{
+    std::ostringstream output;
+
+    cppf::ScalarConverter::write(literal, output);
+    return output.str();
+}
+
+std::size_t countNewlines(const std::string &text)
+{
+    std::size_t count = 0;
+    std::size_t index;
+
+    for (index = 0; index < text.size(); ++index)
+    {
+        if (text[index] == '\n')
+            ++count;
+    }
+    return count;
+}
+
+void testScalarIntegerProperty()
+{
+    std::size_t index;
+
+    for (index = 0; index < 2048; ++index)
+    {
+        const long value =
+            static_cast<long>(nextRandom() % 200001UL) - 100000L;
+        const std::string literal = decimal(value);
+        const std::string rendered = scalarOutput(literal);
+        const std::string expected_line = "int: " + literal + "\n";
+        const std::string second_rendering = scalarOutput(literal);
+
+        check(rendered.find(expected_line) != std::string::npos,
+              "scalar literal=" + literal + " expected_fragment=" +
+                  escaped(expected_line) + " actual=" + escaped(rendered));
+        check(countNewlines(rendered) == 4,
+              "scalar literal=" + literal +
+                  " expected_newlines=4 actual_newlines=" +
+                  decimal(static_cast<long>(countNewlines(rendered))));
+        check(second_rendering == rendered,
+              "scalar literal=" + literal + " expected=" +
+                  escaped(rendered) + " actual=" +
+                  escaped(second_rendering));
+
+        std::ostringstream invalid_output;
+        bool rejected = false;
+
+        try
+        {
+            cppf::ScalarConverter::write(literal + "x", invalid_output);
+        }
+        catch (const cppf::InvalidScalar &)
+        {
+            rejected = true;
+        }
+        check(rejected,
+              "scalar invalid_literal=" + literal +
+                  "x expected_rejected=true actual_rejected=" +
+                  booleanText(rejected));
+        check(invalid_output.str().empty(),
+              "scalar invalid_literal=" + literal +
+                  "x expected_output=<empty> actual_output=" +
+                  escaped(invalid_output.str()));
+    }
+}
+
+char nextOperator()
+{
+    static const char operators[] = {'+', '-', '*', '/'};
+
+    return operators[nextRandom() % 4UL];
+}
+
+void testRpnBinaryProperty()
+{
+    std::size_t index;
+
+    for (index = 0; index < 4096; ++index)
+    {
+        const long left =
+            static_cast<long>(nextRandom() % 2001UL) - 1000L;
+        long right =
+            static_cast<long>(nextRandom() % 2001UL) - 1000L;
+        const char operation = nextOperator();
+        long expected;
+
+        if (operation == '/' && right == 0)
+            right = 1;
+        if (operation == '+')
+            expected = left + right;
+        else if (operation == '-')
+            expected = left - right;
+        else if (operation == '*')
+            expected = left * right;
+        else
+            expected = left / right;
+
+        const std::string expression =
+            decimal(left) + " " + decimal(right) + " " + operation;
+        const long actual = cppf::RpnEvaluator::evaluate(expression);
+
+        check(actual == expected,
+              "rpn expression=" + expression + " expected=" +
+                  decimal(expected) + " actual=" + decimal(actual));
+    }
+}
+
+struct ExpectedJob
+{
+    std::string name;
+    long value;
+};
+
+bool expectedLess(const ExpectedJob &left, const ExpectedJob &right)
+{
+    if (left.value != right.value)
+        return left.value < right.value;
+    return left.name < right.name;
+}
+
+std::string jobName(std::size_t index)
+{
+    std::ostringstream output;
+
+    output << "job_" << std::setw(5) << std::setfill('0') << index
+           << "_payload_abcdefghijklmnop";
+    return output.str();
+}
+
+void testLargeBatchProperty()
+{
+    const std::size_t job_count = 4096;
+    std::ostringstream input_text;
+    std::vector<ExpectedJob> expected;
+    std::size_t index;
+
+    expected.reserve(job_count);
+    for (index = 0; index < job_count; ++index)
+    {
+        const long left =
+            static_cast<long>(nextRandom() % 100000UL);
+        const long right =
+            static_cast<long>(nextRandom() % 100000UL);
+        ExpectedJob job;
+
+        job.name = jobName(index);
+        job.value = left - right;
+        expected.push_back(job);
+        input_text << job.name << " | " << left << ' ' << right << " -";
+        if (index + 1 != job_count)
+            input_text << '\n';
+    }
+    check(input_text.str().size() > 200000U,
+          "batch input expected_bytes>200000 actual_bytes=" +
+              decimal(static_cast<long>(input_text.str().size())));
+
+    std::istringstream input(input_text.str());
+    cppf::BatchEngine engine;
+
+    engine.replace(input);
+    std::sort(expected.begin(), expected.end(), expectedLess);
+    check(engine.results().size() == expected.size(),
+          "batch result count expected=" +
+              decimal(static_cast<long>(expected.size())) + " actual=" +
+              decimal(static_cast<long>(engine.results().size())));
+    for (index = 0; index < expected.size(); ++index)
+    {
+        check(engine.results()[index].name() == expected[index].name,
+              "batch index=" + decimal(static_cast<long>(index)) +
+                  " expected_name=" + expected[index].name +
+                  " actual_name=" + engine.results()[index].name());
+        check(engine.results()[index].value() == expected[index].value,
+              "batch index=" + decimal(static_cast<long>(index)) +
+                  " expected_value=" + decimal(expected[index].value) +
+                  " actual_value=" +
+                  decimal(engine.results()[index].value()));
+    }
+
+    std::ostringstream first_output;
+    std::ostringstream second_output;
+
+    engine.write(first_output);
+    engine.write(second_output);
+    check(first_output.str() == second_output.str(),
+          "batch deterministic output expected_equal=true actual_equal=" +
+              std::string(booleanText(
+                  first_output.str() == second_output.str())));
+    check(first_output.str().size() > 150000U,
+          "batch output expected_bytes>150000 actual_bytes=" +
+              decimal(static_cast<long>(first_output.str().size())));
+}
+
+}
+
+int main()
+{
+    testScalarIntegerProperty();
+    testRpnBinaryProperty();
+    testLargeBatchProperty();
+    if (failures != 0)
+    {
+        std::cerr << failures << " boundary property checks failed\n"
+                  << "seed=" << fixed_seed << '\n'
+                  << "first_counterexample=" << first_failure
+                  << std::endl;
+        return 1;
+    }
+    std::cout << checks << " boundary property checks passed; seed="
+              << fixed_seed << std::endl;
+    return 0;
+}
diff --git a/tests/run_with_timeout.sh b/tests/run_with_timeout.sh
new file mode 100755
index 0000000..3e464c7
--- /dev/null
+++ b/tests/run_with_timeout.sh
@@ -0,0 +1,89 @@
+#!/bin/sh
+
+set -u
+
+if [ "$#" -lt 2 ]
+then
+    printf '제한 시간과 실행할 명령이 필요합니다.\n' >&2
+    exit 2
+fi
+
+case $1 in
+    ''|*[!0-9]*)
+        printf '제한 시간은 양의 정수여야 합니다.\n' >&2
+        exit 2
+        ;;
+esac
+if [ "$1" -eq 0 ]
+then
+    printf '제한 시간은 양의 정수여야 합니다.\n' >&2
+    exit 2
+fi
+
+limit_seconds=$1
+shift
+temporary_directory=$(mktemp -d \
+    "${TMPDIR:-/tmp}/cpp-foundation-timeout.XXXXXX") || exit 2
+timeout_marker=$temporary_directory/timed-out
+child_pid=
+watchdog_pid=
+
+cleanup()
+{
+    if [ -n "$watchdog_pid" ]
+    then
+        kill "$watchdog_pid" 2>/dev/null || true
+        wait "$watchdog_pid" 2>/dev/null || true
+    fi
+    if [ -n "$child_pid" ]
+    then
+        kill "$child_pid" 2>/dev/null || true
+        wait "$child_pid" 2>/dev/null || true
+    fi
+    rm -f "$timeout_marker"
+    rmdir "$temporary_directory"
+}
+
+trap cleanup EXIT
+trap 'exit 129' HUP
+trap 'exit 130' INT
+trap 'exit 143' TERM
+
+started_at=$(date +%s)
+"$@" &
+child_pid=$!
+
+(
+    sleep "$limit_seconds"
+    if kill -0 "$child_pid" 2>/dev/null
+    then
+        : > "$timeout_marker"
+        kill -TERM "$child_pid" 2>/dev/null || true
+        sleep 2
+        kill -KILL "$child_pid" 2>/dev/null || true
+    fi
+) &
+watchdog_pid=$!
+
+if wait "$child_pid"
+then
+    command_status=0
+else
+    command_status=$?
+fi
+child_pid=
+kill "$watchdog_pid" 2>/dev/null || true
+wait "$watchdog_pid" 2>/dev/null || true
+watchdog_pid=
+finished_at=$(date +%s)
+wall_seconds=$((finished_at - started_at))
+
+printf 'wall_seconds=%s timeout_seconds=%s command=%s\n' \
+    "$wall_seconds" "$limit_seconds" "$1"
+
+if [ -f "$timeout_marker" ]
+then
+    printf '제한 시간을 초과했습니다: %s초\n' "$limit_seconds" >&2
+    exit 124
+fi
+exit "$command_status"
