# 오버플로 검사 RPN 평가

## `feat(rpn): signed token과 stack 문법 처리`

diff --git a/include/cppf/RpnEvaluator.hpp b/include/cppf/RpnEvaluator.hpp
new file mode 100644
index 0000000..c834a14
--- /dev/null
+++ b/include/cppf/RpnEvaluator.hpp
@@ -0,0 +1,22 @@
+#ifndef CPPF_RPN_EVALUATOR_HPP
+#define CPPF_RPN_EVALUATOR_HPP
+
+#include <string>
+
+namespace cppf
+{
+
+class RpnEvaluator
+{
+public:
+    static long evaluate(const std::string &expression);
+
+private:
+    RpnEvaluator();
+    RpnEvaluator(const RpnEvaluator &other);
+    RpnEvaluator &operator=(const RpnEvaluator &other);
+};
+
+}
+
+#endif
diff --git a/src/RpnEvaluator.cpp b/src/RpnEvaluator.cpp
new file mode 100644
index 0000000..f4b5ae7
--- /dev/null
+++ b/src/RpnEvaluator.cpp
@@ -0,0 +1,88 @@
+#include "cppf/RpnEvaluator.hpp"
+
+#include <limits>
+#include <stdexcept>
+#include <vector>
+
+namespace
+{
+
+bool isDigit(char value)
+{
+    return value >= '0' && value <= '9';
+}
+
+bool parseLong(const std::string &token, long &value)
+{
+    std::size_t index = 0;
+    bool negative = false;
+    unsigned long magnitude = 0;
+    unsigned long limit;
+
+    if (token.empty())
+        return false;
+    if (token[index] == '+' || token[index] == '-')
+    {
+        negative = token[index] == '-';
+        ++index;
+    }
+    if (index == token.size())
+        return false;
+    limit = static_cast<unsigned long>(std::numeric_limits<long>::max());
+    if (negative)
+        ++limit;
+    while (index < token.size())
+    {
+        unsigned long digit;
+
+        if (!isDigit(token[index]))
+            return false;
+        digit = static_cast<unsigned long>(token[index] - '0');
+        if (magnitude > (limit - digit) / 10)
+            throw std::overflow_error("rpn overflow");
+        magnitude = magnitude * 10 + digit;
+        ++index;
+    }
+    if (!negative)
+        value = static_cast<long>(magnitude);
+    else if (magnitude == limit)
+        value = std::numeric_limits<long>::min();
+    else
+        value = -static_cast<long>(magnitude);
+    return true;
+}
+
+}
+
+namespace cppf
+{
+
+long RpnEvaluator::evaluate(const std::string &expression)
+{
+    std::vector<long> stack;
+    std::size_t index = 0;
+
+    while (index < expression.size())
+    {
+        std::size_t start;
+        long value;
+
+        while (index < expression.size() && expression[index] == ' ')
+            ++index;
+        if (index == expression.size())
+            break;
+        start = index;
+        while (index < expression.size() && expression[index] != ' ')
+            ++index;
+        const std::string token = expression.substr(start, index - start);
+
+        if (!parseLong(token, value))
+            throw std::invalid_argument("invalid rpn expression");
+        stack.push_back(value);
+    }
+    if (stack.size() != 1)
+        throw std::invalid_argument("invalid rpn expression");
+    return stack.back();
+}
+
+}


## `feat(rpn): overflow 검사 산술 연산 구현`

diff --git a/src/RpnEvaluator.cpp b/src/RpnEvaluator.cpp
index f4b5ae7..54f7474 100644
--- a/src/RpnEvaluator.cpp
+++ b/src/RpnEvaluator.cpp
@@ -52,6 +52,83 @@ bool parseLong(const std::string &token, long &value)
     return true;
 }
 
+unsigned long magnitudeOf(long value)
+{
+    if (value >= 0)
+        return static_cast<unsigned long>(value);
+    return static_cast<unsigned long>(-(value + 1)) + 1;
+}
+
+long checkedAdd(long left, long right)
+{
+    if ((right > 0 &&
+         left > std::numeric_limits<long>::max() - right) ||
+        (right < 0 &&
+         left < std::numeric_limits<long>::min() - right))
+        throw std::overflow_error("rpn overflow");
+    return left + right;
+}
+
+long checkedSubtract(long left, long right)
+{
+    if ((right > 0 &&
+         left < std::numeric_limits<long>::min() + right) ||
+        (right < 0 &&
+         left > std::numeric_limits<long>::max() + right))
+        throw std::overflow_error("rpn overflow");
+    return left - right;
+}
+
+long checkedMultiply(long left, long right)
+{
+    const bool negative = (left < 0) != (right < 0);
+    const unsigned long left_magnitude = magnitudeOf(left);
+    const unsigned long right_magnitude = magnitudeOf(right);
+    unsigned long limit =
+        static_cast<unsigned long>(std::numeric_limits<long>::max());
+    unsigned long product;
+
+    if (left_magnitude == 0 || right_magnitude == 0)
+        return 0;
+    if (negative)
+        ++limit;
+    if (left_magnitude > limit / right_magnitude)
+        throw std::overflow_error("rpn overflow");
+    product = left_magnitude * right_magnitude;
+    if (!negative)
+        return static_cast<long>(product);
+    if (product == limit)
+        return std::numeric_limits<long>::min();
+    return -static_cast<long>(product);
+}
+
+long checkedDivide(long left, long right)
+{
+    if (right == 0)
+        throw std::invalid_argument("invalid rpn expression");
+    if (left == std::numeric_limits<long>::min() && right == -1)
+        throw std::overflow_error("rpn overflow");
+    return left / right;
+}
+
+long applyOperator(long left, long right, char operation)
+{
+    if (operation == '+')
+        return checkedAdd(left, right);
+    if (operation == '-')
+        return checkedSubtract(left, right);
+    if (operation == '*')
+        return checkedMultiply(left, right);
+    return checkedDivide(left, right);
+}
+
+bool isOperator(const std::string &token)
+{
+    return token.size() == 1 &&
+           (token[0] == '+' || token[0] == '-' || token[0] == '*' ||
+            token[0] == '/');
+}
+
 }
 
 namespace cppf
@@ -76,9 +153,23 @@ long RpnEvaluator::evaluate(const std::string &expression)
             ++index;
         const std::string token = expression.substr(start, index - start);
 
-        if (!parseLong(token, value))
+        if (isOperator(token))
+        {
+            long right;
+            long left;
+
+            if (stack.size() < 2)
+                throw std::invalid_argument("invalid rpn expression");
+            right = stack.back();
+            stack.pop_back();
+            left = stack.back();
+            stack.pop_back();
+            stack.push_back(applyOperator(left, right, token[0]));
+        }
+        else if (parseLong(token, value))
+            stack.push_back(value);
+        else
             throw std::invalid_argument("invalid rpn expression");
-        stack.push_back(value);
     }
     if (stack.size() != 1)
         throw std::invalid_argument("invalid rpn expression");


## `test(rpn): 산술 경계와 잘못된 token 검증`

diff --git a/Makefile b/Makefile
index 36d96d3..389fb9c 100644
--- a/Makefile
+++ b/Makefile
@@ -88,6 +88,8 @@ test-contract:
 		tests/compile/serializer_headers.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/template_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/rpn_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/compile/rpn_headers.cpp b/tests/compile/rpn_headers.cpp
new file mode 100644
index 0000000..2bd669e
--- /dev/null
+++ b/tests/compile/rpn_headers.cpp
@@ -0,0 +1,7 @@
+#include "cppf/RpnEvaluator.hpp"
+#include "cppf/RpnEvaluator.hpp"
+
+int main()
+{
+    return cppf::RpnEvaluator::evaluate("2 3 +") != 5;
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 921ae42..e7ca77d 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -11,6 +11,7 @@ void testScalarConverter(test_support::Suite &suite);
 void testRuntimeType(test_support::Suite &suite);
 void testSerializer(test_support::Suite &suite);
 void testRandomAccessBatch(test_support::Suite &suite);
+void testRpnEvaluator(test_support::Suite &suite);
 
 int main()
 {
@@ -27,5 +28,6 @@ int main()
     testRuntimeType(suite);
     testSerializer(suite);
     testRandomAccessBatch(suite);
+    testRpnEvaluator(suite);
     return suite.result();
 }
diff --git a/tests/test_rpn_evaluator.cpp b/tests/test_rpn_evaluator.cpp
new file mode 100644
index 0000000..30f916b
--- /dev/null
+++ b/tests/test_rpn_evaluator.cpp
@@ -0,0 +1,143 @@
+#include "cppf/RpnEvaluator.hpp"
+#include "support/Test.hpp"
+
+#include <limits>
+#include <sstream>
+#include <stdexcept>
+#include <string>
+
+namespace
+{
+
+std::string decimal(long value)
+{
+    std::ostringstream output;
+
+    output << value;
+    return output.str();
+}
+
+std::string unsignedDecimal(unsigned long value)
+{
+    std::ostringstream output;
+
+    output << value;
+    return output.str();
+}
+
+void checkInvalid(test_support::Suite &suite,
+                  const std::string &expression,
+                  const char *label)
+{
+    bool invalid = false;
+
+    try
+    {
+        cppf::RpnEvaluator::evaluate(expression);
+    }
+    catch (const std::invalid_argument &error)
+    {
+        invalid = std::string(error.what()) == "invalid rpn expression";
+    }
+    suite.check(invalid, label);
+}
+
+void checkOverflow(test_support::Suite &suite,
+                   const std::string &expression,
+                   const char *label)
+{
+    bool overflow = false;
+
+    try
+    {
+        cppf::RpnEvaluator::evaluate(expression);
+    }
+    catch (const std::overflow_error &error)
+    {
+        overflow = std::string(error.what()) == "rpn overflow";
+    }
+    suite.check(overflow, label);
+}
+
+}
+
+void testRpnEvaluator(test_support::Suite &suite)
+{
+    suite.check(cppf::RpnEvaluator::evaluate("3 4 +") == 7,
+                "rpn adds operands");
+    suite.check(cppf::RpnEvaluator::evaluate("9 4 -") == 5,
+                "rpn preserves subtraction order");
+    suite.check(cppf::RpnEvaluator::evaluate("6 -3 *") == -18,
+                "rpn multiplies signed operands");
+    suite.check(cppf::RpnEvaluator::evaluate("-7 2 /") == -3,
+                "rpn division truncates toward zero");
+    suite.check(cppf::RpnEvaluator::evaluate("  2  3 + 4 *  ") == 20,
+                "rpn accepts repeated ascii space separators");
+    suite.check(cppf::RpnEvaluator::evaluate("+2 -3 * 8 +") == 2,
+                "rpn parses explicit numeric signs");
+
+    const long maximum = std::numeric_limits<long>::max();
+    const long minimum = std::numeric_limits<long>::min();
+    const unsigned long minimum_magnitude =
+        static_cast<unsigned long>(maximum) + 1;
+
+    suite.check(cppf::RpnEvaluator::evaluate(decimal(maximum)) == maximum,
+                "rpn parses long maximum");
+    suite.check(cppf::RpnEvaluator::evaluate(decimal(minimum)) == minimum,
+                "rpn parses long minimum");
+    checkOverflow(suite, unsignedDecimal(minimum_magnitude),
+                  "rpn rejects positive literal above long maximum");
+    checkOverflow(suite, "-" + unsignedDecimal(minimum_magnitude + 1),
+                  "rpn rejects negative literal below long minimum");
+    checkOverflow(suite, decimal(maximum) + " 1 +",
+                  "rpn rejects addition overflow");
+    checkOverflow(suite, decimal(minimum) + " 1 -",
+                  "rpn rejects subtraction overflow");
+    checkOverflow(suite, decimal(maximum) + " 2 *",
+                  "rpn rejects multiplication overflow");
+    checkOverflow(suite, decimal(minimum) + " -1 /",
+                  "rpn rejects division overflow");
+    suite.check(cppf::RpnEvaluator::evaluate(decimal(maximum) + " 0 +") ==
+                    maximum,
+                "rpn preserves maximum when adding zero");
+    suite.check(cppf::RpnEvaluator::evaluate(decimal(minimum) + " 0 -") ==
+                    minimum,
+                "rpn preserves minimum when subtracting zero");
+    checkOverflow(suite, decimal(minimum) + " -1 +",
+                  "rpn rejects negative addition underflow");
+    checkOverflow(suite, decimal(maximum) + " -1 -",
+                  "rpn rejects negative subtraction overflow");
+    suite.check(cppf::RpnEvaluator::evaluate("3 4 *") == 12 &&
+                    cppf::RpnEvaluator::evaluate("3 -4 *") == -12 &&
+                    cppf::RpnEvaluator::evaluate("-3 4 *") == -12 &&
+                    cppf::RpnEvaluator::evaluate("-3 -4 *") == 12,
+                "rpn multiplication covers every sign combination");
+    suite.check(cppf::RpnEvaluator::evaluate(
+                    decimal(minimum) + " 1 *") == minimum,
+                "rpn multiplication reaches long minimum exactly");
+    suite.check(cppf::RpnEvaluator::evaluate(decimal(minimum) + " 1 /") ==
+                    minimum,
+                "rpn division preserves long minimum by one");
+    suite.check(cppf::RpnEvaluator::evaluate("0007") == 7 &&
+                    cppf::RpnEvaluator::evaluate("+0") == 0 &&
+                    cppf::RpnEvaluator::evaluate("-0") == 0,
+                "rpn accepts decimal zero forms");
+
+    checkInvalid(suite, "", "rpn rejects empty expression");
+    checkInvalid(suite, "   ", "rpn rejects spaces-only expression");
+    checkInvalid(suite, "1 0 /", "rpn rejects division by zero");
+    checkInvalid(suite, "1 +", "rpn rejects missing operand");
+    checkInvalid(suite, "1 2", "rpn rejects leftover operands");
+    checkInvalid(suite, "1 2 %", "rpn rejects unknown operator");
+    checkInvalid(suite, "--2", "rpn rejects malformed signed number");
+    checkInvalid(suite, "2x", "rpn rejects trailing numeric bytes");
+    checkInvalid(suite, "2.0", "rpn rejects decimal point");
+    checkInvalid(suite, "2e1", "rpn rejects exponent syntax");
+    checkInvalid(suite, "2f", "rpn rejects numeric suffix");
+    checkInvalid(suite, "2\t3 +", "rpn rejects tab separator");
+    checkInvalid(suite, "2 3 +\n", "rpn rejects newline separator");
+    checkInvalid(suite, std::string("2\0", 2),
+                 "rpn rejects embedded nul");
+    checkInvalid(suite, std::string(1, static_cast<char>(0x80)),
+                 "rpn rejects non-ascii byte");
+}


