# 등록형 런타임 타입 식별

## `feat(rtti): 다형 객체의 실행 시간 타입 식별`

diff --git a/include/cppf/RuntimeType.hpp b/include/cppf/RuntimeType.hpp
new file mode 100644
index 0000000..fb209a4
--- /dev/null
+++ b/include/cppf/RuntimeType.hpp
@@ -0,0 +1,52 @@
+#ifndef CPPF_RUNTIME_TYPE_HPP
+#define CPPF_RUNTIME_TYPE_HPP
+
+namespace cppf
+{
+
+enum RuntimeKind
+{
+    runtime_a,
+    runtime_b,
+    runtime_c,
+    runtime_unknown
+};
+
+class RuntimeBase
+{
+public:
+    virtual ~RuntimeBase();
+
+protected:
+    RuntimeBase();
+};
+
+class RuntimeA : public RuntimeBase
+{
+};
+
+class RuntimeB : public RuntimeBase
+{
+};
+
+class RuntimeC : public RuntimeBase
+{
+};
+
+class RuntimeInspector
+{
+public:
+    static RuntimeBase *create(RuntimeKind kind);
+    static RuntimeKind identify(const RuntimeBase *value);
+    static RuntimeKind identify(const RuntimeBase &value);
+    static const char *name(RuntimeKind kind);
+
+private:
+    RuntimeInspector();
+    RuntimeInspector(const RuntimeInspector &other);
+    RuntimeInspector &operator=(const RuntimeInspector &other);
+};
+
+}
+
+#endif
diff --git a/src/RuntimeType.cpp b/src/RuntimeType.cpp
new file mode 100644
index 0000000..566b6fa
--- /dev/null
+++ b/src/RuntimeType.cpp
@@ -0,0 +1,81 @@
+#include "cppf/RuntimeType.hpp"
+
+#include <typeinfo>
+
+namespace cppf
+{
+
+RuntimeBase::RuntimeBase()
+{
+}
+
+RuntimeBase::~RuntimeBase()
+{
+}
+
+RuntimeBase *RuntimeInspector::create(RuntimeKind kind)
+{
+    if (kind == runtime_a)
+        return new RuntimeA();
+    if (kind == runtime_b)
+        return new RuntimeB();
+    if (kind == runtime_c)
+        return new RuntimeC();
+    return 0;
+}
+
+RuntimeKind RuntimeInspector::identify(const RuntimeBase *value)
+{
+    if (dynamic_cast<const RuntimeA *>(value) != 0)
+        return runtime_a;
+    if (dynamic_cast<const RuntimeB *>(value) != 0)
+        return runtime_b;
+    if (dynamic_cast<const RuntimeC *>(value) != 0)
+        return runtime_c;
+    return runtime_unknown;
+}
+
+RuntimeKind RuntimeInspector::identify(const RuntimeBase &value)
+{
+    try
+    {
+        const RuntimeA &matched = dynamic_cast<const RuntimeA &>(value);
+        static_cast<void>(matched);
+        return runtime_a;
+    }
+    catch (const std::bad_cast &)
+    {
+    }
+    try
+    {
+        const RuntimeB &matched = dynamic_cast<const RuntimeB &>(value);
+        static_cast<void>(matched);
+        return runtime_b;
+    }
+    catch (const std::bad_cast &)
+    {
+    }
+    try
+    {
+        const RuntimeC &matched = dynamic_cast<const RuntimeC &>(value);
+        static_cast<void>(matched);
+        return runtime_c;
+    }
+    catch (const std::bad_cast &)
+    {
+    }
+    return runtime_unknown;
+}
+
+const char *RuntimeInspector::name(RuntimeKind kind)
+{
+    if (kind == runtime_a)
+        return "A";
+    if (kind == runtime_b)
+        return "B";
+    if (kind == runtime_c)
+        return "C";
+    return "unknown";
+}
+
+}


## `test(rtti): pointer·reference 식별 경계 검증`

diff --git a/Makefile b/Makefile
index f45428d..2743d38 100644
--- a/Makefile
+++ b/Makefile
@@ -82,6 +82,8 @@ test-contract:
 		tests/compile/format_headers.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/scalar_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/compile/runtime_headers.cpp b/tests/compile/runtime_headers.cpp
new file mode 100644
index 0000000..c244d7b
--- /dev/null
+++ b/tests/compile/runtime_headers.cpp
@@ -0,0 +1,13 @@
+#include "cppf/RuntimeType.hpp"
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    cppf::RuntimeBase *value =
+        cppf::RuntimeInspector::create(cppf::runtime_a);
+    const cppf::RuntimeKind kind =
+        cppf::RuntimeInspector::identify(value);
+
+    delete value;
+    return kind != cppf::runtime_a;
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 23e0496..3958b29 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -8,6 +8,7 @@ void testFormatPipeline(test_support::Suite &suite);
 void testFactory(test_support::Suite &suite);
 void testScalarLiteral(test_support::Suite &suite);
 void testScalarConverter(test_support::Suite &suite);
+void testRuntimeType(test_support::Suite &suite);
 
 int main()
 {
@@ -21,5 +22,6 @@ int main()
     testFactory(suite);
     testScalarLiteral(suite);
     testScalarConverter(suite);
+    testRuntimeType(suite);
     return suite.result();
 }
diff --git a/tests/test_runtime_type.cpp b/tests/test_runtime_type.cpp
new file mode 100644
index 0000000..14d6ae5
--- /dev/null
+++ b/tests/test_runtime_type.cpp
@@ -0,0 +1,98 @@
+#include "cppf/RuntimeType.hpp"
+#include "support/Test.hpp"
+
+#include <cstddef>
+#include <cstring>
+
+namespace
+{
+
+class UnknownRuntime : public cppf::RuntimeBase
+{
+};
+
+class TrackedRuntime : public cppf::RuntimeBase
+{
+public:
+    explicit TrackedRuntime(bool &destroyed) : destroyed_(destroyed)
+    {
+    }
+
+    virtual ~TrackedRuntime()
+    {
+        destroyed_ = true;
+    }
+
+private:
+    TrackedRuntime(const TrackedRuntime &other);
+    TrackedRuntime &operator=(const TrackedRuntime &other);
+
+    bool &destroyed_;
+};
+
+}
+
+void testRuntimeType(test_support::Suite &suite)
+{
+    const cppf::RuntimeA value_a;
+    const cppf::RuntimeB value_b;
+    const cppf::RuntimeC value_c;
+    const UnknownRuntime unknown;
+
+    suite.check(cppf::RuntimeInspector::identify(&value_a) ==
+                    cppf::runtime_a,
+                "runtime pointer identifies A");
+    suite.check(cppf::RuntimeInspector::identify(value_a) ==
+                    cppf::runtime_a,
+                "runtime reference identifies A");
+    suite.check(cppf::RuntimeInspector::identify(&value_b) ==
+                    cppf::runtime_b &&
+                    cppf::RuntimeInspector::identify(value_b) ==
+                    cppf::runtime_b,
+                "runtime identifies B overloads");
+    suite.check(cppf::RuntimeInspector::identify(&value_c) ==
+                    cppf::runtime_c &&
+                    cppf::RuntimeInspector::identify(value_c) ==
+                    cppf::runtime_c,
+                "runtime identifies C overloads");
+    suite.check(cppf::RuntimeInspector::identify(
+                    static_cast<const cppf::RuntimeBase *>(0)) ==
+                    cppf::runtime_unknown,
+                "runtime pointer identifies null as unknown");
+    suite.check(cppf::RuntimeInspector::identify(&unknown) ==
+                    cppf::runtime_unknown &&
+                    cppf::RuntimeInspector::identify(unknown) ==
+                    cppf::runtime_unknown,
+                "runtime overloads reject unregistered derived type");
+
+    const cppf::RuntimeKind kinds[] = {
+        cppf::runtime_a, cppf::runtime_b, cppf::runtime_c};
+    std::size_t index;
+    bool created = true;
+
+    for (index = 0; index < 3; ++index)
+    {
+        cppf::RuntimeBase *value =
+            cppf::RuntimeInspector::create(kinds[index]);
+
+        if (value == 0 || cppf::RuntimeInspector::identify(*value) !=
+                              kinds[index])
+            created = false;
+        delete value;
+    }
+    suite.check(created, "runtime factory creates every registered type");
+    suite.check(cppf::RuntimeInspector::create(cppf::runtime_unknown) == 0,
+                "runtime factory rejects unknown kind");
+    suite.check(std::strcmp(cppf::RuntimeInspector::name(cppf::runtime_a),
+                            "A") == 0 &&
+                    std::strcmp(
+                        cppf::RuntimeInspector::name(cppf::runtime_unknown),
+                        "unknown") == 0,
+                "runtime names are stable");
+
+    bool destroyed = false;
+    cppf::RuntimeBase *tracked = new TrackedRuntime(destroyed);
+
+    delete tracked;
+    suite.check(destroyed, "runtime base deletion dispatches destructor");
+}


## `feat(casts): runtime type CLI mode 추가`

diff --git a/apps/ex04_type_boundary.cpp b/apps/ex04_type_boundary.cpp
index 130d6c7..dc482c4 100644
--- a/apps/ex04_type_boundary.cpp
+++ b/apps/ex04_type_boundary.cpp
@@ -1,17 +1,30 @@
 #include "cppf/ScalarConverter.hpp"
+#include "cppf/RuntimeType.hpp"
 
 #include <iostream>
+#include <string>
 
-int main(int argument_count, char **arguments)
+namespace
+{
+
+bool parseRuntimeKind(const std::string &text, cppf::RuntimeKind &kind)
+{
+    if (text == "A")
+        kind = cppf::runtime_a;
+    else if (text == "B")
+        kind = cppf::runtime_b;
+    else if (text == "C")
+        kind = cppf::runtime_c;
+    else
+        return false;
+    return true;
+}
+
+int runScalar(const char *literal)
 {
-    if (argument_count != 3 || std::string(arguments[1]) != "scalar")
-    {
-        std::cerr << "usage: ex04_type_boundary scalar LITERAL" << std::endl;
-        return 1;
-    }
     try
     {
-        cppf::ScalarConverter::write(arguments[2], std::cout);
+        cppf::ScalarConverter::write(literal, std::cout);
     }
     catch (const cppf::InvalidScalar &error)
     {
@@ -20,3 +33,54 @@ int main(int argument_count, char **arguments)
     }
     return 0;
 }
+
+int runRuntime(const char *name)
+{
+    cppf::RuntimeKind kind;
+
+    if (!parseRuntimeKind(name, kind))
+    {
+        std::cerr << "unknown runtime kind" << std::endl;
+        return 1;
+    }
+    cppf::RuntimeBase *value = cppf::RuntimeInspector::create(kind);
+    const cppf::RuntimeKind pointer_kind =
+        cppf::RuntimeInspector::identify(value);
+    const cppf::RuntimeKind reference_kind =
+        cppf::RuntimeInspector::identify(*value);
+
+    delete value;
+    std::cout << "pointer: " << cppf::RuntimeInspector::name(pointer_kind)
+              << '\n';
+    std::cout << "reference: "
+              << cppf::RuntimeInspector::name(reference_kind) << '\n';
+    return 0;
+}
+
+}
+
+int main(int argument_count, char **arguments)
+{
+    if (argument_count >= 2 && std::string(arguments[1]) == "scalar")
+    {
+        if (argument_count != 3)
+        {
+            std::cerr << "usage: ex04_type_boundary scalar LITERAL"
+                      << std::endl;
+            return 1;
+        }
+        return runScalar(arguments[2]);
+    }
+    if (argument_count >= 2 && std::string(arguments[1]) == "runtime")
+    {
+        if (argument_count != 3)
+        {
+            std::cerr << "usage: ex04_type_boundary runtime A|B|C"
+                      << std::endl;
+            return 1;
+        }
+        return runRuntime(arguments[2]);
+    }
+    std::cerr << "usage: ex04_type_boundary MODE ..." << std::endl;
+    return 1;
+}


## `test(casts): 타입·주소 변환의 공개 경계 검증`

diff --git a/Makefile b/Makefile
index 4ca9b2f..a29f156 100644
--- a/Makefile
+++ b/Makefile
@@ -90,6 +90,14 @@ test-contract:
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/formatter_abstract_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_inspector_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_unrelated_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/serializer_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/serializer_const_fail.cpp >/dev/null 2>&1
 
 test-integration: $(APP_BIN)
 	sh tests/check_cli.sh
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
index 18d08ba..6069752 100644
--- a/tests/check_cli.sh
+++ b/tests/check_cli.sh
@@ -52,3 +52,51 @@ printf 'invalid scalar literal\n' \
     > "$temporary_directory/scalar-failure.expected"
 diff -u "$temporary_directory/scalar-failure.expected" \
     "$temporary_directory/scalar-failure.err"
+
+./bin/ex04_type_boundary runtime A \
+    > "$temporary_directory/runtime.out"
+printf 'pointer: A\nreference: A\n' \
+    > "$temporary_directory/runtime.expected"
+diff -u "$temporary_directory/runtime.expected" \
+    "$temporary_directory/runtime.out"
+
+./bin/ex04_type_boundary address 42 alpha \
+    > "$temporary_directory/address.out"
+printf 'token: nonzero\nsame: yes\nid: 42\nlabel: alpha\n' \
+    > "$temporary_directory/address.expected"
+diff -u "$temporary_directory/address.expected" \
+    "$temporary_directory/address.out"
+
+if ./bin/ex04_type_boundary runtime Z \
+    > "$temporary_directory/runtime-failure.out" \
+    2> "$temporary_directory/runtime-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/runtime-failure.out"
+printf 'unknown runtime kind\n' \
+    > "$temporary_directory/runtime-failure.expected"
+diff -u "$temporary_directory/runtime-failure.expected" \
+    "$temporary_directory/runtime-failure.err"
+
+if ./bin/ex04_type_boundary address 42x alpha \
+    > "$temporary_directory/address-failure.out" \
+    2> "$temporary_directory/address-failure.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/address-failure.out"
+printf 'invalid payload id\n' \
+    > "$temporary_directory/address-failure.expected"
+diff -u "$temporary_directory/address-failure.expected" \
+    "$temporary_directory/address-failure.err"
+
+if ./bin/ex04_type_boundary address 18446744073709551616 alpha \
+    > "$temporary_directory/address-overflow.out" \
+    2> "$temporary_directory/address-overflow.err"
+then
+    exit 1
+fi
+test ! -s "$temporary_directory/address-overflow.out"
+diff -u "$temporary_directory/address-failure.expected" \
+    "$temporary_directory/address-overflow.err"
diff --git a/tests/compile/runtime_inspector_private_fail.cpp b/tests/compile/runtime_inspector_private_fail.cpp
new file mode 100644
index 0000000..a142f7e
--- /dev/null
+++ b/tests/compile/runtime_inspector_private_fail.cpp
@@ -0,0 +1,9 @@
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    cppf::RuntimeInspector inspector;
+    return cppf::RuntimeInspector::identify(
+               static_cast<const cppf::RuntimeBase *>(0)) ==
+           cppf::runtime_unknown;
+}
diff --git a/tests/compile/runtime_unrelated_fail.cpp b/tests/compile/runtime_unrelated_fail.cpp
new file mode 100644
index 0000000..0ec158a
--- /dev/null
+++ b/tests/compile/runtime_unrelated_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    int value = 0;
+    return cppf::RuntimeInspector::identify(&value);
+}
diff --git a/tests/compile/serializer_const_fail.cpp b/tests/compile/serializer_const_fail.cpp
new file mode 100644
index 0000000..251488f
--- /dev/null
+++ b/tests/compile/serializer_const_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/Serializer.hpp"
+
+int main()
+{
+    const cppf::Payload payload(7, "value");
+    return cppf::Serializer::serialize(&payload) == 0;
+}
diff --git a/tests/compile/serializer_private_fail.cpp b/tests/compile/serializer_private_fail.cpp
new file mode 100644
index 0000000..1e8d2ab
--- /dev/null
+++ b/tests/compile/serializer_private_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/Serializer.hpp"
+
+int main()
+{
+    cppf::Serializer serializer;
+    return 0;
+}
diff --git a/tests/test_runtime_type.cpp b/tests/test_runtime_type.cpp
index 14d6ae5..dde3035 100644
--- a/tests/test_runtime_type.cpp
+++ b/tests/test_runtime_type.cpp
@@ -30,6 +30,10 @@ private:
     bool &destroyed_;
 };
 
+class KnownChild : public cppf::RuntimeA
+{
+};
+
 }
 
 void testRuntimeType(test_support::Suite &suite)
@@ -83,6 +87,12 @@ void testRuntimeType(test_support::Suite &suite)
     suite.check(created, "runtime factory creates every registered type");
     suite.check(cppf::RuntimeInspector::create(cppf::runtime_unknown) == 0,
                 "runtime factory rejects unknown kind");
+    const cppf::RuntimeKind invalid_kind =
+        static_cast<cppf::RuntimeKind>(999);
+    suite.check(cppf::RuntimeInspector::create(invalid_kind) == 0 &&
+                    std::strcmp(cppf::RuntimeInspector::name(invalid_kind),
+                                "unknown") == 0,
+                "runtime rejects invalid enum value");
     suite.check(std::strcmp(cppf::RuntimeInspector::name(cppf::runtime_a),
                             "A") == 0 &&
                     std::strcmp(
@@ -90,6 +100,24 @@ void testRuntimeType(test_support::Suite &suite)
                         "unknown") == 0,
                 "runtime names are stable");
 
+    const KnownChild known_child;
+    suite.check(cppf::RuntimeInspector::identify(&known_child) ==
+                    cppf::runtime_a &&
+                    cppf::RuntimeInspector::identify(known_child) ==
+                    cppf::runtime_a,
+                "runtime recognizes a registered subtype descendant");
+
+    bool escaped = false;
+    try
+    {
+        cppf::RuntimeInspector::identify(unknown);
+    }
+    catch (...)
+    {
+        escaped = true;
+    }
+    suite.check(!escaped, "runtime reference hides bad cast failures");
+
     bool destroyed = false;
     cppf::RuntimeBase *tracked = new TrackedRuntime(destroyed);
 


## `test(rtti): integer에서 runtime kind로의 암시 변환 거부`

diff --git a/Makefile b/Makefile
index 8feb601..21cb3ec 100644
--- a/Makefile
+++ b/Makefile
@@ -122,6 +122,8 @@ test-contract:
 		tests/compile/runtime_inspector_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/runtime_unrelated_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_integer_kind_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/serializer_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/compile/runtime_integer_kind_fail.cpp b/tests/compile/runtime_integer_kind_fail.cpp
new file mode 100644
index 0000000..cae16f6
--- /dev/null
+++ b/tests/compile/runtime_integer_kind_fail.cpp
@@ -0,0 +1,9 @@
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    cppf::RuntimeBase *value = cppf::RuntimeInspector::create(999);
+
+    delete value;
+    return 0;
+}
diff --git a/tests/test_runtime_type.cpp b/tests/test_runtime_type.cpp
index dde3035..29850ef 100644
--- a/tests/test_runtime_type.cpp
+++ b/tests/test_runtime_type.cpp
@@ -87,12 +87,6 @@ void testRuntimeType(test_support::Suite &suite)
     suite.check(created, "runtime factory creates every registered type");
     suite.check(cppf::RuntimeInspector::create(cppf::runtime_unknown) == 0,
                 "runtime factory rejects unknown kind");
-    const cppf::RuntimeKind invalid_kind =
-        static_cast<cppf::RuntimeKind>(999);
-    suite.check(cppf::RuntimeInspector::create(invalid_kind) == 0 &&
-                    std::strcmp(cppf::RuntimeInspector::name(invalid_kind),
-                                "unknown") == 0,
-                "runtime rejects invalid enum value");
     suite.check(std::strcmp(cppf::RuntimeInspector::name(cppf::runtime_a),
                             "A") == 0 &&
                     std::strcmp(
