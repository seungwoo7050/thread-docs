# 스칼라 리터럴 문법과 수치 경계

## `feat(scalar): scalar 리터럴 문법과 종류 분류`

diff --git a/src/ScalarLiteral.cpp b/src/ScalarLiteral.cpp
new file mode 100644
index 0000000..ad8296a
--- /dev/null
+++ b/src/ScalarLiteral.cpp
@@ -0,0 +1,134 @@
+#include "ScalarLiteral.hpp"
+
+#include <limits>
+#include <locale>
+#include <sstream>
+
+namespace
+{
+
+bool isDigit(char value)
+{
+    return value >= '0' && value <= '9';
+}
+
+bool isWhitespace(char value)
+{
+    return value == ' ' || value == '\t' || value == '\n' ||
+           value == '\v' || value == '\f' || value == '\r';
+}
+
+void rejectInvalidBytes(const std::string &text)
+{
+    if (text.empty())
+        throw cppf::scalar_detail::ScalarParseError();
+    for (std::size_t index = 0; index < text.size(); ++index)
+    {
+        const unsigned char value =
+            static_cast<unsigned char>(text[index]);
+
+        if (value == 0 || value > 127 || isWhitespace(text[index]))
+            throw cppf::scalar_detail::ScalarParseError();
+    }
+}
+
+void validateFiniteGrammar(const std::string &text, bool &float_suffix)
+{
+    std::size_t index = 0;
+    std::size_t mantissa_digits = 0;
+
+    if (text[index] == '+' || text[index] == '-')
+        ++index;
+    while (index < text.size() && isDigit(text[index]))
+    {
+        ++mantissa_digits;
+        ++index;
+    }
+    if (index < text.size() && text[index] == '.')
+    {
+        ++index;
+        while (index < text.size() && isDigit(text[index]))
+        {
+            ++mantissa_digits;
+            ++index;
+        }
+    }
+    if (mantissa_digits == 0)
+        throw cppf::scalar_detail::ScalarParseError();
+    if (index < text.size() && (text[index] == 'e' || text[index] == 'E'))
+    {
+        std::size_t exponent_digits = 0;
+
+        ++index;
+        if (index < text.size() &&
+            (text[index] == '+' || text[index] == '-'))
+            ++index;
+        while (index < text.size() && isDigit(text[index]))
+        {
+            ++exponent_digits;
+            ++index;
+        }
+        if (exponent_digits == 0)
+            throw cppf::scalar_detail::ScalarParseError();
+    }
+    float_suffix = index < text.size() && text[index] == 'f';
+    if (float_suffix)
+        ++index;
+    if (index != text.size())
+        throw cppf::scalar_detail::ScalarParseError();
+}
+
+}
+
+namespace cppf
+{
+namespace scalar_detail
+{
+
+ScalarLiteral parseScalarLiteral(const std::string &text)
+{
+    ScalarLiteral literal;
+
+    rejectInvalidBytes(text);
+    literal.float_suffix = text[text.size() - 1] == 'f';
+    literal.negative_zero = false;
+    if (text == "nan" || text == "nanf")
+    {
+        literal.kind = literal_nan;
+        literal.value = std::numeric_limits<double>::quiet_NaN();
+        return literal;
+    }
+    if (text == "+inf" || text == "+inff" ||
+        text == "-inf" || text == "-inff")
+    {
+        literal.kind = text[0] == '-' ? literal_negative_infinity
+                                      : literal_positive_infinity;
+        literal.value = text[0] == '-'
+                            ? -std::numeric_limits<double>::infinity()
+                            : std::numeric_limits<double>::infinity();
+        return literal;
+    }
+    if (text.size() == 1 && !isDigit(text[0]))
+    {
+        literal.kind = literal_character;
+        literal.value = static_cast<unsigned char>(text[0]);
+        literal.float_suffix = false;
+        return literal;
+    }
+    validateFiniteGrammar(text, literal.float_suffix);
+    const std::string number = literal.float_suffix
+                                   ? text.substr(0, text.size() - 1)
+                                   : text;
+    std::istringstream input(number);
+
+    input.imbue(std::locale::classic());
+    input >> literal.value;
+    if (input.fail() || !input.eof())
+        throw ScalarParseError();
+    literal.kind = literal_finite;
+    literal.negative_zero = text[0] == '-' && literal.value == 0.0;
+    return literal;
+}
+
+}
+}
diff --git a/src/ScalarLiteral.hpp b/src/ScalarLiteral.hpp
new file mode 100644
index 0000000..ff8c47c
--- /dev/null
+++ b/src/ScalarLiteral.hpp
@@ -0,0 +1,37 @@
+#ifndef CPP_FOUNDATION_SCALAR_LITERAL_HPP
+#define CPP_FOUNDATION_SCALAR_LITERAL_HPP
+
+#include <string>
+
+namespace cppf
+{
+namespace scalar_detail
+{
+
+enum LiteralKind
+{
+    literal_character,
+    literal_finite,
+    literal_nan,
+    literal_positive_infinity,
+    literal_negative_infinity
+};
+
+struct ScalarLiteral
+{
+    LiteralKind kind;
+    double value;
+    bool float_suffix;
+    bool negative_zero;
+};
+
+class ScalarParseError
+{
+};
+
+ScalarLiteral parseScalarLiteral(const std::string &text);
+
+}
+}
+
+#endif


## `feat(scalar): locale 고정 수치 추출과 경계 보존`

diff --git a/src/ScalarLiteral.cpp b/src/ScalarLiteral.cpp
index ad8296a..4c6f2ed 100644
--- a/src/ScalarLiteral.cpp
+++ b/src/ScalarLiteral.cpp
@@ -20,9 +20,11 @@ bool isWhitespace(char value)
 
 void rejectInvalidBytes(const std::string &text)
 {
+    std::size_t index;
+
     if (text.empty())
         throw cppf::scalar_detail::ScalarParseError();
-    for (std::size_t index = 0; index < text.size(); ++index)
+    for (index = 0; index < text.size(); ++index)
     {
         const unsigned char value =
             static_cast<unsigned char>(text[index]);
@@ -32,33 +34,72 @@ void rejectInvalidBytes(const std::string &text)
     }
 }
 
+cppf::scalar_detail::ScalarLiteral makeSpecial(
+    cppf::scalar_detail::LiteralKind kind,
+    bool float_suffix)
+{
+    cppf::scalar_detail::ScalarLiteral literal;
+
+    literal.kind = kind;
+    literal.float_suffix = float_suffix;
+    literal.negative_zero = false;
+    if (kind == cppf::scalar_detail::literal_nan)
+        literal.value = std::numeric_limits<double>::quiet_NaN();
+    else if (kind == cppf::scalar_detail::literal_negative_infinity)
+        literal.value = -std::numeric_limits<double>::infinity();
+    else
+        literal.value = std::numeric_limits<double>::infinity();
+    return literal;
+}
+
+bool allMantissaDigitsAreZero(const std::string &text)
+{
+    std::size_t index = 0;
+
+    if (text[index] == '+' || text[index] == '-')
+        ++index;
+    while (index < text.size() && text[index] != 'e' &&
+           text[index] != 'E' && text[index] != 'f')
+    {
+        if (isDigit(text[index]) && text[index] != '0')
+            return false;
+        ++index;
+    }
+    return true;
+}
+
 void validateFiniteGrammar(const std::string &text, bool &float_suffix)
 {
     std::size_t index = 0;
-    std::size_t mantissa_digits = 0;
+    std::size_t integer_digits = 0;
+    std::size_t fraction_digits = 0;
+    bool has_point = false;
+    bool has_exponent = false;
 
     if (text[index] == '+' || text[index] == '-')
         ++index;
     while (index < text.size() && isDigit(text[index]))
     {
-        ++mantissa_digits;
+        ++integer_digits;
         ++index;
     }
     if (index < text.size() && text[index] == '.')
     {
+        has_point = true;
         ++index;
         while (index < text.size() && isDigit(text[index]))
         {
-            ++mantissa_digits;
+            ++fraction_digits;
             ++index;
         }
     }
-    if (mantissa_digits == 0)
+    if (integer_digits == 0 && fraction_digits == 0)
         throw cppf::scalar_detail::ScalarParseError();
     if (index < text.size() && (text[index] == 'e' || text[index] == 'E'))
     {
         std::size_t exponent_digits = 0;
 
+        has_exponent = true;
         ++index;
         if (index < text.size() &&
             (text[index] == '+' || text[index] == '-'))
@@ -71,13 +112,32 @@ void validateFiniteGrammar(const std::string &text, bool &float_suffix)
         if (exponent_digits == 0)
             throw cppf::scalar_detail::ScalarParseError();
     }
-    float_suffix = index < text.size() && text[index] == 'f';
-    if (float_suffix)
+    float_suffix = false;
+    if (index < text.size() && text[index] == 'f')
+    {
+        float_suffix = true;
         ++index;
-    if (index != text.size())
+    }
+    if (index != text.size() || (float_suffix && !has_point && !has_exponent))
         throw cppf::scalar_detail::ScalarParseError();
 }
 
+double extractFiniteValue(const std::string &text, bool float_suffix)
+{
+    const std::string number =
+        float_suffix ? text.substr(0, text.size() - 1) : text;
+    std::istringstream input(number);
+    double value;
+
+    input.imbue(std::locale::classic());
+    input >> value;
+    if (input.fail() || !input.eof() || value != value ||
+        value > std::numeric_limits<double>::max() ||
+        value < -std::numeric_limits<double>::max())
+        throw cppf::scalar_detail::ScalarParseError();
+    return value;
+}
+
 }
 
 namespace cppf
@@ -88,45 +148,38 @@ namespace scalar_detail
 ScalarLiteral parseScalarLiteral(const std::string &text)
 {
     ScalarLiteral literal;
+    bool float_suffix;
+    bool all_zero;
 
     rejectInvalidBytes(text);
-    literal.float_suffix = text[text.size() - 1] == 'f';
-    literal.negative_zero = false;
     if (text == "nan" || text == "nanf")
-    {
-        literal.kind = literal_nan;
-        literal.value = std::numeric_limits<double>::quiet_NaN();
-        return literal;
-    }
-    if (text == "+inf" || text == "+inff" ||
-        text == "-inf" || text == "-inff")
-    {
-        literal.kind = text[0] == '-' ? literal_negative_infinity
-                                      : literal_positive_infinity;
-        literal.value = text[0] == '-'
-                            ? -std::numeric_limits<double>::infinity()
-                            : std::numeric_limits<double>::infinity();
-        return literal;
-    }
-    if (text.size() == 1 && !isDigit(text[0]))
+        return makeSpecial(literal_nan, text == "nanf");
+    if (text == "+inf" || text == "+inff")
+        return makeSpecial(literal_positive_infinity, text == "+inff");
+    if (text == "-inf" || text == "-inff")
+        return makeSpecial(literal_negative_infinity, text == "-inff");
+    if (text.size() == 1 && !isDigit(text[0]) &&
+        static_cast<unsigned char>(text[0]) >= 33)
     {
         literal.kind = literal_character;
         literal.value = static_cast<unsigned char>(text[0]);
         literal.float_suffix = false;
+        literal.negative_zero = false;
         return literal;
     }
-    validateFiniteGrammar(text, literal.float_suffix);
-    const std::string number = literal.float_suffix
-                                   ? text.substr(0, text.size() - 1)
-                                   : text;
-    std::istringstream input(number);
-
-    input.imbue(std::locale::classic());
-    input >> literal.value;
-    if (input.fail() || !input.eof())
-        throw ScalarParseError();
+    validateFiniteGrammar(text, float_suffix);
+    all_zero = allMantissaDigitsAreZero(text);
     literal.kind = literal_finite;
-    literal.negative_zero = text[0] == '-' && literal.value == 0.0;
+    literal.float_suffix = float_suffix;
+    literal.negative_zero = text[0] == '-' && all_zero;
+    if (all_zero)
+        literal.value = literal.negative_zero ? -0.0 : 0.0;
+    else
+    {
+        literal.value = extractFiniteValue(text, float_suffix);
+        if (literal.value == 0.0)
+            throw ScalarParseError();
+    }
     return literal;
 }
 


## `test(scalar): literal 문법과 수치 범위 검증`

diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 0183fcf..f7c0429 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -6,6 +6,7 @@ void testTextBuffer(test_support::Suite &suite);
 void testFormatter(test_support::Suite &suite);
 void testFormatPipeline(test_support::Suite &suite);
 void testFactory(test_support::Suite &suite);
+void testScalarLiteral(test_support::Suite &suite);
 
 int main()
 {
@@ -17,5 +18,6 @@ int main()
     testFormatter(suite);
     testFormatPipeline(suite);
     testFactory(suite);
+    testScalarLiteral(suite);
     return suite.result();
 }
diff --git a/tests/test_scalar_literal.cpp b/tests/test_scalar_literal.cpp
new file mode 100644
index 0000000..81c959b
--- /dev/null
+++ b/tests/test_scalar_literal.cpp
@@ -0,0 +1,143 @@
+#include "../src/ScalarLiteral.hpp"
+#include "support/Test.hpp"
+
+#include <string>
+
+namespace
+{
+
+void checkInvalid(test_support::Suite &suite,
+                  const std::string &text,
+                  const char *label)
+{
+    bool threw = false;
+
+    try
+    {
+        cppf::scalar_detail::parseScalarLiteral(text);
+    }
+    catch (const cppf::scalar_detail::ScalarParseError &)
+    {
+        threw = true;
+    }
+    suite.check(threw, label);
+}
+
+void checkCharacter(test_support::Suite &suite,
+                    const char *text,
+                    int expected,
+                    const char *label)
+{
+    const cppf::scalar_detail::ScalarLiteral literal =
+        cppf::scalar_detail::parseScalarLiteral(text);
+
+    suite.check(literal.kind == cppf::scalar_detail::literal_character &&
+                    literal.value == expected,
+                label);
+}
+
+void checkFinite(test_support::Suite &suite,
+                 const char *text,
+                 double expected,
+                 bool float_suffix,
+                 const char *label)
+{
+    const cppf::scalar_detail::ScalarLiteral literal =
+        cppf::scalar_detail::parseScalarLiteral(text);
+
+    suite.check(literal.kind == cppf::scalar_detail::literal_finite &&
+                    literal.value == expected &&
+                    literal.float_suffix == float_suffix,
+                label);
+}
+
+void checkNegativeZero(test_support::Suite &suite,
+                       const char *text,
+                       const char *label)
+{
+    const cppf::scalar_detail::ScalarLiteral literal =
+        cppf::scalar_detail::parseScalarLiteral(text);
+
+    suite.check(literal.kind == cppf::scalar_detail::literal_finite &&
+                    literal.value == 0.0 && literal.negative_zero,
+                label);
+}
+
+void checkSpecial(test_support::Suite &suite,
+                  const char *text,
+                  cppf::scalar_detail::LiteralKind kind,
+                  bool float_suffix,
+                  const char *label)
+{
+    const cppf::scalar_detail::ScalarLiteral literal =
+        cppf::scalar_detail::parseScalarLiteral(text);
+
+    suite.check(literal.kind == kind &&
+                    literal.float_suffix == float_suffix,
+                label);
+}
+
+}
+
+void testScalarLiteral(test_support::Suite &suite)
+{
+    checkCharacter(suite, "a", 97, "scalar parses alphabetic character");
+    checkCharacter(suite, "f", 102, "scalar prefers lone f character");
+    checkCharacter(suite, "+", 43, "scalar parses lone plus character");
+    checkCharacter(suite, "-", 45, "scalar parses lone minus character");
+    checkCharacter(suite, ".", 46, "scalar parses lone point character");
+
+    checkFinite(suite, "0", 0.0, false, "scalar prefers zero number");
+    checkFinite(suite, "9", 9.0, false, "scalar prefers digit number");
+    checkFinite(suite, "+42", 42.0, false, "scalar parses positive integer");
+    checkFinite(suite, "-42", -42.0, false, "scalar parses negative integer");
+    checkFinite(suite, "42.", 42.0, false, "scalar parses trailing point");
+    checkFinite(suite, ".5", 0.5, false, "scalar parses leading point");
+    checkFinite(suite, "1.e2", 100.0, false, "scalar parses point exponent");
+    checkFinite(suite, "1e2", 100.0, false, "scalar parses exponent");
+    checkFinite(suite, "1e2f", 100.0, true, "scalar parses float exponent");
+
+    checkNegativeZero(suite, "-0", "scalar preserves integer negative zero");
+    checkNegativeZero(suite, "-0.0", "scalar preserves decimal negative zero");
+    checkNegativeZero(suite, "-0e20", "scalar preserves exponent negative zero");
+    checkNegativeZero(suite, "-0.0f", "scalar preserves float negative zero");
+
+    checkSpecial(suite, "nan", cppf::scalar_detail::literal_nan, false,
+                 "scalar parses double nan");
+    checkSpecial(suite, "nanf", cppf::scalar_detail::literal_nan, true,
+                 "scalar parses float nan");
+    checkSpecial(suite, "+inf",
+                 cppf::scalar_detail::literal_positive_infinity, false,
+                 "scalar parses positive double infinity");
+    checkSpecial(suite, "+inff",
+                 cppf::scalar_detail::literal_positive_infinity, true,
+                 "scalar parses positive float infinity");
+    checkSpecial(suite, "-inf",
+                 cppf::scalar_detail::literal_negative_infinity, false,
+                 "scalar parses negative double infinity");
+    checkSpecial(suite, "-inff",
+                 cppf::scalar_detail::literal_negative_infinity, true,
+                 "scalar parses negative float infinity");
+
+    checkInvalid(suite, "", "scalar rejects empty input");
+    checkInvalid(suite, " ", "scalar rejects space");
+    checkInvalid(suite, "\t42", "scalar rejects leading tab");
+    checkInvalid(suite, "42\n", "scalar rejects trailing newline");
+    checkInvalid(suite, "4 2", "scalar rejects embedded space");
+    checkInvalid(suite, "++1", "scalar rejects repeated sign");
+    checkInvalid(suite, "..", "scalar rejects repeated point");
+    checkInvalid(suite, "1e", "scalar rejects empty exponent");
+    checkInvalid(suite, "1e+", "scalar rejects signed empty exponent");
+    checkInvalid(suite, "42f", "scalar rejects suffix without float syntax");
+    checkInvalid(suite, "42F", "scalar rejects uppercase suffix");
+    checkInvalid(suite, "0x10", "scalar rejects hexadecimal syntax");
+    checkInvalid(suite, "1,5", "scalar rejects locale decimal comma");
+    checkInvalid(suite, "42x", "scalar rejects trailing garbage");
+    checkInvalid(suite, "nanx", "scalar rejects extended nan");
+    checkInvalid(suite, "inf", "scalar rejects unsigned infinity");
+    checkInvalid(suite, "+nan", "scalar rejects signed nan");
+    checkInvalid(suite, std::string("4\0", 2),
+                 "scalar rejects embedded nul");
+    checkInvalid(suite, "1e309", "scalar rejects double overflow");
+    checkInvalid(suite, "1e-9999", "scalar rejects nonzero underflow");
+}
