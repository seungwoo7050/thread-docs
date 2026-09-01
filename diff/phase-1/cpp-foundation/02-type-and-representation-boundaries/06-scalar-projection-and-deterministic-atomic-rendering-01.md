# 스칼라 투영과 결정적 원자 출력

## `feat(scalar): 문자와 정수 투영 결과 출력`

diff --git a/include/cppf/ScalarConverter.hpp b/include/cppf/ScalarConverter.hpp
new file mode 100644
index 0000000..c5aa74d
--- /dev/null
+++ b/include/cppf/ScalarConverter.hpp
@@ -0,0 +1,30 @@
+#ifndef CPPF_SCALAR_CONVERTER_HPP
+#define CPPF_SCALAR_CONVERTER_HPP
+
+#include <exception>
+#include <iosfwd>
+#include <string>
+
+namespace cppf
+{
+
+class InvalidScalar : public std::exception
+{
+public:
+    virtual const char *what() const throw();
+};
+
+class ScalarConverter
+{
+public:
+    static void write(const std::string &literal, std::ostream &output);
+
+private:
+    ScalarConverter();
+    ScalarConverter(const ScalarConverter &other);
+    ScalarConverter &operator=(const ScalarConverter &other);
+};
+
+}
+
+#endif
diff --git a/src/ScalarConverter.cpp b/src/ScalarConverter.cpp
new file mode 100644
index 0000000..e1abd94
--- /dev/null
+++ b/src/ScalarConverter.cpp
@@ -0,0 +1,98 @@
+#include "cppf/ScalarConverter.hpp"
+
+#include "ScalarLiteral.hpp"
+
+#include <limits>
+#include <ostream>
+
+namespace
+{
+
+bool isValue(const cppf::scalar_detail::ScalarLiteral &literal)
+{
+    return literal.kind == cppf::scalar_detail::literal_character ||
+           literal.kind == cppf::scalar_detail::literal_finite;
+}
+
+bool canProjectChar(const cppf::scalar_detail::ScalarLiteral &literal)
+{
+    return isValue(literal) && literal.value > -1.0 &&
+           literal.value < 128.0;
+}
+
+bool canProjectInt(const cppf::scalar_detail::ScalarLiteral &literal)
+{
+    const double lower =
+        static_cast<double>(std::numeric_limits<int>::min()) - 1.0;
+    const double upper =
+        static_cast<double>(std::numeric_limits<int>::max()) + 1.0;
+
+    return isValue(literal) && literal.value > lower &&
+           literal.value < upper;
+}
+
+std::string quotedCharacter(int value)
+{
+    if (value == '\'')
+        return "'\\''";
+    if (value == '\\')
+        return "'\\\\'";
+    return std::string("'") + static_cast<char>(value) + "'";
+}
+
+void writeCharacter(const cppf::scalar_detail::ScalarLiteral &literal,
+                    std::ostream &output)
+{
+    output << "char: ";
+    if (!canProjectChar(literal))
+        output << "impossible";
+    else
+    {
+        const int value = static_cast<int>(literal.value);
+
+        if (value < 32 || value > 126)
+            output << "Non displayable";
+        else
+            output << quotedCharacter(value);
+    }
+    output << '\n';
+}
+
+void writeInteger(const cppf::scalar_detail::ScalarLiteral &literal,
+                  std::ostream &output)
+{
+    output << "int: ";
+    if (!canProjectInt(literal))
+        output << "impossible";
+    else
+        output << static_cast<int>(literal.value);
+    output << '\n';
+}
+
+}
+
+namespace cppf
+{
+
+const char *InvalidScalar::what() const throw()
+{
+    return "invalid scalar literal";
+}
+
+void ScalarConverter::write(const std::string &text, std::ostream &output)
+{
+    scalar_detail::ScalarLiteral literal;
+
+    try
+    {
+        literal = scalar_detail::parseScalarLiteral(text);
+    }
+    catch (const scalar_detail::ScalarParseError &)
+    {
+        throw InvalidScalar();
+    }
+    writeCharacter(literal, output);
+    writeInteger(literal, output);
+}
+
+}


## `feat(scalar): 부동소수점 표현과 원자 출력 구현`

diff --git a/src/ScalarConverter.cpp b/src/ScalarConverter.cpp
index e1abd94..4f87c67 100644
--- a/src/ScalarConverter.cpp
+++ b/src/ScalarConverter.cpp
@@ -3,7 +3,9 @@
 #include "ScalarLiteral.hpp"
 
 #include <limits>
+#include <locale>
 #include <ostream>
+#include <sstream>
 
 namespace
 {
@@ -69,6 +71,78 @@ void writeInteger(const cppf::scalar_detail::ScalarLiteral &literal,
     output << '\n';
 }
 
+std::string finiteNumber(double value, bool as_float, bool negative_zero)
+{
+    std::ostringstream output;
+    std::string result;
+
+    output.imbue(std::locale::classic());
+    output.precision(as_float ? std::numeric_limits<float>::digits10
+                              : std::numeric_limits<double>::digits10);
+    if (value == 0.0 && negative_zero)
+        result = "-0";
+    else if (as_float)
+        output << static_cast<float>(value);
+    else
+        output << value;
+    if (result.empty())
+        result = output.str();
+    if (result.find('.') == std::string::npos &&
+        result.find('e') == std::string::npos &&
+        result.find('E') == std::string::npos)
+        result += ".0";
+    return result;
+}
+
+bool canProjectFloat(const cppf::scalar_detail::ScalarLiteral &literal)
+{
+    const double maximum = std::numeric_limits<float>::max();
+    float value;
+
+    if (!isValue(literal) || literal.value < -maximum ||
+        literal.value > maximum)
+        return false;
+    value = static_cast<float>(literal.value);
+    return literal.value == 0.0 || value != 0.0f;
+}
+
+void writeFloating(const cppf::scalar_detail::ScalarLiteral &literal,
+                   std::ostream &output)
+{
+    output << "float: ";
+    if (literal.kind == cppf::scalar_detail::literal_nan)
+        output << "nanf";
+    else if (literal.kind ==
+             cppf::scalar_detail::literal_positive_infinity)
+        output << "+inff";
+    else if (literal.kind ==
+             cppf::scalar_detail::literal_negative_infinity)
+        output << "-inff";
+    else if (!canProjectFloat(literal))
+        output << "impossible";
+    else
+        output << finiteNumber(literal.value, true, literal.negative_zero)
+               << 'f';
+    output << '\n';
+}
+
+void writeDouble(const cppf::scalar_detail::ScalarLiteral &literal,
+                 std::ostream &output)
+{
+    output << "double: ";
+    if (literal.kind == cppf::scalar_detail::literal_nan)
+        output << "nan";
+    else if (literal.kind ==
+             cppf::scalar_detail::literal_positive_infinity)
+        output << "+inf";
+    else if (literal.kind ==
+             cppf::scalar_detail::literal_negative_infinity)
+        output << "-inf";
+    else
+        output << finiteNumber(literal.value, false, literal.negative_zero);
+    output << '\n';
+}
+
 }
 
 namespace cppf
@@ -91,8 +165,16 @@ void ScalarConverter::write(const std::string &text, std::ostream &output)
     {
         throw InvalidScalar();
     }
-    writeCharacter(literal, output);
-    writeInteger(literal, output);
+    std::ostringstream rendered;
+
+    rendered.imbue(std::locale::classic());
+    writeCharacter(literal, rendered);
+    writeInteger(literal, rendered);
+    writeFloating(literal, rendered);
+    writeDouble(literal, rendered);
+    const std::string result = rendered.str();
+
+    output.write(result.data(), static_cast<std::streamsize>(result.size()));
 }
 
 }


## `feat(scalar): type boundary CLI의 scalar mode 제공`

diff --git a/apps/ex04_type_boundary.cpp b/apps/ex04_type_boundary.cpp
new file mode 100644
index 0000000..130d6c7
--- /dev/null
+++ b/apps/ex04_type_boundary.cpp
@@ -0,0 +1,22 @@
+#include "cppf/ScalarConverter.hpp"
+
+#include <iostream>
+
+int main(int argument_count, char **arguments)
+{
+    if (argument_count != 3 || std::string(arguments[1]) != "scalar")
+    {
+        std::cerr << "usage: ex04_type_boundary scalar LITERAL" << std::endl;
+        return 1;
+    }
+    try
+    {
+        cppf::ScalarConverter::write(arguments[2], std::cout);
+    }
+    catch (const cppf::InvalidScalar &error)
+    {
+        std::cerr << error.what() << std::endl;
+        return 1;
+    }
+    return 0;
+}


## `test(scalar): 변환 가능성·출력·CLI 오류 검증`

diff --git a/Makefile b/Makefile
index 47551ad..f45428d 100644
--- a/Makefile
+++ b/Makefile
@@ -80,6 +80,8 @@ test-contract:
 		tests/compile/contact_headers.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/format_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/scalar_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index 56973df..18d08ba 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -33,3 +33,22 @@ test ! -s "$temporary_directory/factory-failure.out"
 printf 'unknown formatter\n' > "$temporary_directory/factory-failure.expected"
 diff -u "$temporary_directory/factory-failure.expected" \
     "$temporary_directory/factory-failure.err"
+
+./bin/ex04_type_boundary scalar 42.5 \
+    > "$temporary_directory/scalar.out"
+printf "char: '*'\nint: 42\nfloat: 42.5f\ndouble: 42.5\n" \
+    > "$temporary_directory/scalar.expected"
+diff -u "$temporary_directory/scalar.expected" \
+    "$temporary_directory/scalar.out"
+
+if ./bin/ex04_type_boundary scalar 42f \
+    > "$temporary_directory/scalar-failure.out" \
+    2> "$temporary_directory/scalar-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/scalar-failure.out"
+printf 'invalid scalar literal\n' \
+    > "$temporary_directory/scalar-failure.expected"
+diff -u "$temporary_directory/scalar-failure.expected" \
+    "$temporary_directory/scalar-failure.err"
diff --git a/tests/compile/scalar_headers.cpp b/tests/compile/scalar_headers.cpp
new file mode 100644
index 0000000..a2d09b2
--- /dev/null
+++ b/tests/compile/scalar_headers.cpp
@@ -0,0 +1,12 @@
+#include "cppf/ScalarConverter.hpp"
+#include "cppf/ScalarConverter.hpp"
+
+#include <sstream>
+
+int main()
+{
+    std::ostringstream output;
+
+    cppf::ScalarConverter::write("42", output);
+    return output.str().empty();
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index f7c0429..23e0496 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -7,6 +7,7 @@ void testFormatter(test_support::Suite &suite);
 void testFormatPipeline(test_support::Suite &suite);
 void testFactory(test_support::Suite &suite);
 void testScalarLiteral(test_support::Suite &suite);
+void testScalarConverter(test_support::Suite &suite);
 
 int main()
 {
@@ -19,5 +20,6 @@ int main()
     testFormatPipeline(suite);
     testFactory(suite);
     testScalarLiteral(suite);
+    testScalarConverter(suite);
     return suite.result();
 }
diff --git a/tests/test_scalar_converter.cpp b/tests/test_scalar_converter.cpp
new file mode 100644
index 0000000..1ba9849
--- /dev/null
+++ b/tests/test_scalar_converter.cpp
@@ -0,0 +1,181 @@
+#include "cppf/ScalarConverter.hpp"
+#include "support/Test.hpp"
+
+#include <ios>
+#include <locale>
+#include <sstream>
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
+std::string convert(const std::string &literal)
+{
+    std::ostringstream output;
+
+    cppf::ScalarConverter::write(literal, output);
+    return output.str();
+}
+
+void checkExact(test_support::Suite &suite,
+                const char *literal,
+                const char *expected,
+                const char *label)
+{
+    suite.check(convert(literal) == expected, label);
+}
+
+}
+
+void testScalarConverter(test_support::Suite &suite)
+{
+    checkExact(suite, "a",
+               "char: 'a'\nint: 97\nfloat: 97.0f\ndouble: 97.0\n",
+               "scalar writes character projections");
+    checkExact(suite, "9",
+               "char: Non displayable\nint: 9\nfloat: 9.0f\n"
+               "double: 9.0\n",
+               "scalar writes digit projections");
+    checkExact(suite, "42.5",
+               "char: '*'\nint: 42\nfloat: 42.5f\ndouble: 42.5\n",
+               "scalar writes fractional projections");
+    checkExact(suite, "-0",
+               "char: Non displayable\nint: 0\nfloat: -0.0f\n"
+               "double: -0.0\n",
+               "scalar preserves negative zero output");
+
+    checkExact(suite, "31",
+               "char: Non displayable\nint: 31\nfloat: 31.0f\n"
+               "double: 31.0\n",
+               "scalar marks ascii control as non-displayable");
+    checkExact(suite, "32",
+               "char: ' '\nint: 32\nfloat: 32.0f\ndouble: 32.0\n",
+               "scalar writes ascii space");
+    checkExact(suite, "39",
+               "char: '\\''\nint: 39\nfloat: 39.0f\ndouble: 39.0\n",
+               "scalar escapes ascii quote");
+    checkExact(suite, "92",
+               "char: '\\\\'\nint: 92\nfloat: 92.0f\ndouble: 92.0\n",
+               "scalar escapes ascii backslash");
+    checkExact(suite, "126",
+               "char: '~'\nint: 126\nfloat: 126.0f\ndouble: 126.0\n",
+               "scalar writes final printable ascii");
+    checkExact(suite, "127",
+               "char: Non displayable\nint: 127\nfloat: 127.0f\n"
+               "double: 127.0\n",
+               "scalar marks ascii delete as non-displayable");
+    checkExact(suite, "128",
+               "char: impossible\nint: 128\nfloat: 128.0f\ndouble: 128.0\n",
+               "scalar rejects out-of-range ascii projection");
+
+    suite.check(convert("-0.5").find("char: Non displayable\nint: 0\n") == 0,
+                "scalar truncates negative fraction toward zero");
+    suite.check(convert("-1").find("char: impossible\nint: -1\n") == 0,
+                "scalar rejects negative character projection");
+    suite.check(convert("127.9").find(
+                    "char: Non displayable\nint: 127\n") == 0,
+                "scalar truncates final ascii fraction");
+    suite.check(convert("2147483647.9").find(
+                    "char: impossible\nint: 2147483647\n") == 0,
+                "scalar accepts upper fractional int edge");
+    suite.check(convert("-2147483648.9").find(
+                    "char: impossible\nint: -2147483648\n") == 0,
+                "scalar accepts lower fractional int edge");
+    suite.check(convert("2147483648.0").find(
+                    "char: impossible\nint: impossible\n") == 0,
+                "scalar rejects value above int range");
+    suite.check(convert("-2147483649.0").find(
+                    "char: impossible\nint: impossible\n") == 0,
+                "scalar rejects value below int range");
+
+    checkExact(suite, "nanf",
+               "char: impossible\nint: impossible\nfloat: nanf\ndouble: nan\n",
+               "scalar canonicalizes nan");
+    checkExact(suite, "+inf",
+               "char: impossible\nint: impossible\nfloat: +inff\n"
+               "double: +inf\n",
+               "scalar canonicalizes positive infinity");
+    checkExact(suite, "-inff",
+               "char: impossible\nint: impossible\nfloat: -inff\n"
+               "double: -inf\n",
+               "scalar canonicalizes negative infinity");
+
+    suite.check(convert("3.402823669209385e38").find(
+                    "float: impossible\n") != std::string::npos,
+                "scalar rejects float overflow only");
+    suite.check(convert("1e-50").find(
+                    "float: impossible\ndouble: 1e-50\n") !=
+                    std::string::npos,
+                "scalar rejects float underflow only");
+    checkExact(suite, "1.23456789",
+               "char: Non displayable\nint: 1\nfloat: 1.23457f\n"
+               "double: 1.23456789\n",
+               "scalar fixes float and double precision");
+
+    std::ostringstream configured;
+    configured.setf(std::ios::scientific, std::ios::floatfield);
+    configured.setf(std::ios::showpos);
+    configured.setf(std::ios::left, std::ios::adjustfield);
+    configured.precision(2);
+    configured.fill('#');
+    configured.width(80);
+    const std::ios::fmtflags original_flags = configured.flags();
+    const std::streamsize original_precision = configured.precision();
+    const char original_fill = configured.fill();
+    const std::streamsize original_width = configured.width();
+
+    cppf::ScalarConverter::write("42.5", configured);
+    suite.check(configured.str() ==
+                    "char: '*'\nint: 42\nfloat: 42.5f\ndouble: 42.5\n",
+                "scalar ignores caller formatting state");
+    suite.check(configured.flags() == original_flags &&
+                    configured.precision() == original_precision &&
+                    configured.fill() == original_fill &&
+                    configured.width() == original_width,
+                "scalar preserves caller formatting state");
+
+    std::ostringstream localized;
+    localized.imbue(std::locale(std::locale::classic(),
+                                new CommaPunctuation));
+    cppf::ScalarConverter::write("42.5", localized);
+    suite.check(localized.str().find("42.5f") != std::string::npos &&
+                    localized.str().find("42,5") == std::string::npos,
+                "scalar fixes classic decimal punctuation");
+
+    std::ostringstream invalid_output;
+    bool invalid = false;
+
+    try
+    {
+        cppf::ScalarConverter::write("42f", invalid_output);
+    }
+    catch (const cppf::InvalidScalar &error)
+    {
+        invalid = std::string(error.what()) == "invalid scalar literal";
+    }
+    suite.check(invalid, "scalar exposes stable invalid literal error");
+    suite.check(invalid_output.str().empty(),
+                "invalid scalar writes no partial output");
+
+    const std::string four_lines = convert("42");
+    std::size_t newlines = 0;
+    std::size_t index;
+
+    for (index = 0; index < four_lines.size(); ++index)
+    {
+        if (four_lines[index] == '\n')
+            ++newlines;
+    }
+    suite.check(newlines == 4 && !four_lines.empty() &&
+                    four_lines[four_lines.size() - 1] == '\n',
+                "scalar always writes four terminated lines");
+}


