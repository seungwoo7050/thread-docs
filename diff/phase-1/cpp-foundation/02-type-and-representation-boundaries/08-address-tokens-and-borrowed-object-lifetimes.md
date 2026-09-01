# 주소 토큰과 차용 객체 수명

## `feat(serialization): 빌린 객체 주소를 token으로 왕복`

diff --git a/include/cppf/Serializer.hpp b/include/cppf/Serializer.hpp
new file mode 100644
index 0000000..0d385fe
--- /dev/null
+++ b/include/cppf/Serializer.hpp
@@ -0,0 +1,34 @@
+#ifndef CPPF_SERIALIZER_HPP
+#define CPPF_SERIALIZER_HPP
+
+#include <stdint.h>
+#include <string>
+
+namespace cppf
+{
+
+struct Payload
+{
+    Payload(unsigned long id_value, const std::string &label_value);
+
+    unsigned long id;
+    std::string label;
+};
+
+class Serializer
+{
+public:
+    typedef uintptr_t raw_type;
+
+    static raw_type serialize(Payload *payload);
+    static Payload *deserialize(raw_type raw);
+
+private:
+    Serializer();
+    Serializer(const Serializer &other);
+    Serializer &operator=(const Serializer &other);
+};
+
+}
+
+#endif
diff --git a/src/Serializer.cpp b/src/Serializer.cpp
new file mode 100644
index 0000000..5f1ef15
--- /dev/null
+++ b/src/Serializer.cpp
@@ -0,0 +1,25 @@
+#include "cppf/Serializer.hpp"
+
+namespace cppf
+{
+
+Payload::Payload(unsigned long id_value, const std::string &label_value)
+    : id(id_value), label(label_value)
+{
+}
+
+Serializer::raw_type Serializer::serialize(Payload *payload)
+{
+    if (payload == 0)
+        return static_cast<raw_type>(0);
+    return reinterpret_cast<raw_type>(payload);
+}
+
+Payload *Serializer::deserialize(raw_type raw)
+{
+    if (raw == static_cast<raw_type>(0))
+        return 0;
+    return reinterpret_cast<Payload *>(raw);
+}
+
+}


## `test(serialization): null과 주소 동일성 검증`

diff --git a/Makefile b/Makefile
index 2743d38..4ca9b2f 100644
--- a/Makefile
+++ b/Makefile
@@ -84,6 +84,8 @@ test-contract:
 		tests/compile/scalar_headers.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/runtime_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/serializer_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/compile/serializer_headers.cpp b/tests/compile/serializer_headers.cpp
new file mode 100644
index 0000000..c77ea39
--- /dev/null
+++ b/tests/compile/serializer_headers.cpp
@@ -0,0 +1,11 @@
+#include "cppf/Serializer.hpp"
+#include "cppf/Serializer.hpp"
+
+int main()
+{
+    cppf::Payload payload(7, "value");
+    const cppf::Serializer::raw_type raw =
+        cppf::Serializer::serialize(&payload);
+
+    return cppf::Serializer::deserialize(raw) != &payload;
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 3958b29..c9686bc 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -9,6 +9,7 @@ void testFactory(test_support::Suite &suite);
 void testScalarLiteral(test_support::Suite &suite);
 void testScalarConverter(test_support::Suite &suite);
 void testRuntimeType(test_support::Suite &suite);
+void testSerializer(test_support::Suite &suite);
 
 int main()
 {
@@ -23,5 +24,6 @@ int main()
     testScalarLiteral(suite);
     testScalarConverter(suite);
     testRuntimeType(suite);
+    testSerializer(suite);
     return suite.result();
 }
diff --git a/tests/test_serializer.cpp b/tests/test_serializer.cpp
new file mode 100644
index 0000000..3111e9d
--- /dev/null
+++ b/tests/test_serializer.cpp
@@ -0,0 +1,54 @@
+#include "cppf/Serializer.hpp"
+#include "support/Test.hpp"
+
+void testSerializer(test_support::Suite &suite)
+{
+    cppf::Payload first(42, "alpha");
+    cppf::Payload second(7, "beta");
+
+    suite.check(first.id == 42 && first.label == "alpha",
+                "payload constructor preserves fields");
+    suite.check(sizeof(cppf::Serializer::raw_type) >=
+                    sizeof(cppf::Payload *),
+                "address token is wide enough for payload pointer");
+
+    const cppf::Serializer::raw_type first_raw =
+        cppf::Serializer::serialize(&first);
+    const cppf::Serializer::raw_type second_raw =
+        cppf::Serializer::serialize(&second);
+    cppf::Payload *recovered = cppf::Serializer::deserialize(first_raw);
+
+    suite.check(first_raw != 0 && recovered == &first,
+                "serializer preserves stack pointer identity");
+    recovered->label = "changed";
+    suite.check(first.label == "changed",
+                "deserialized pointer aliases the live payload");
+    suite.check(first_raw != second_raw &&
+                    cppf::Serializer::deserialize(second_raw) == &second,
+                "serializer distinguishes simultaneous live payloads");
+
+    cppf::Payload *heap = new cppf::Payload(99, "heap");
+    const cppf::Serializer::raw_type heap_raw =
+        cppf::Serializer::serialize(heap);
+    cppf::Payload *heap_recovered =
+        cppf::Serializer::deserialize(heap_raw);
+
+    suite.check(heap_recovered == heap && heap_recovered->id == 99,
+                "serializer round-trips heap payload during its lifetime");
+    delete heap;
+
+    suite.check(cppf::Serializer::serialize(0) == 0 &&
+                    cppf::Serializer::deserialize(0) == 0,
+                "serializer preserves null token");
+
+    cppf::Serializer::raw_type expired;
+    {
+        cppf::Payload scoped(5, "scoped");
+
+        expired = cppf::Serializer::serialize(&scoped);
+        suite.check(cppf::Serializer::deserialize(expired) == &scoped,
+                    "address token is valid inside payload lifetime");
+    }
+    suite.check(expired != 0,
+                "expired token remains opaque and is not dereferenced");
+}


## `feat(casts): address token CLI mode 추가`

diff --git a/apps/ex04_type_boundary.cpp b/apps/ex04_type_boundary.cpp
index dc482c4..fcff046 100644
--- a/apps/ex04_type_boundary.cpp
+++ b/apps/ex04_type_boundary.cpp
@@ -1,7 +1,9 @@
 #include "cppf/ScalarConverter.hpp"
 #include "cppf/RuntimeType.hpp"
+#include "cppf/Serializer.hpp"
 
 #include <iostream>
+#include <limits>
 #include <string>
 
 namespace
@@ -20,6 +22,28 @@ bool parseRuntimeKind(const std::string &text, cppf::RuntimeKind &kind)
     return true;
 }
 
+bool parsePayloadId(const std::string &text, unsigned long &value)
+{
+    std::size_t index;
+
+    if (text.empty())
+        return false;
+    value = 0;
+    for (index = 0; index < text.size(); ++index)
+    {
+        unsigned long digit;
+
+        if (text[index] < '0' || text[index] > '9')
+            return false;
+        digit = static_cast<unsigned long>(text[index] - '0');
+        if (value > (std::numeric_limits<unsigned long>::max() - digit) /
+                        10)
+            return false;
+        value = value * 10 + digit;
+    }
+    return true;
+}
+
 int runScalar(const char *literal)
 {
     try
@@ -57,6 +81,28 @@ int runRuntime(const char *name)
     return 0;
 }
 
+int runAddress(const char *id_text, const char *label)
+{
+    unsigned long id;
+
+    if (!parsePayloadId(id_text, id))
+    {
+        std::cerr << "invalid payload id" << std::endl;
+        return 1;
+    }
+    cppf::Payload payload(id, label);
+    const cppf::Serializer::raw_type token =
+        cppf::Serializer::serialize(&payload);
+    cppf::Payload *recovered = cppf::Serializer::deserialize(token);
+
+    std::cout << "token: " << (token == 0 ? "zero" : "nonzero") << '\n';
+    std::cout << "same: " << (recovered == &payload ? "yes" : "no")
+              << '\n';
+    std::cout << "id: " << recovered->id << '\n';
+    std::cout << "label: " << recovered->label << '\n';
+    return 0;
+}
+
 }
 
 int main(int argument_count, char **arguments)
@@ -81,6 +127,16 @@ int main(int argument_count, char **arguments)
         }
         return runRuntime(arguments[2]);
     }
+    if (argument_count >= 2 && std::string(arguments[1]) == "address")
+    {
+        if (argument_count != 4)
+        {
+            std::cerr << "usage: ex04_type_boundary address ID LABEL"
+                      << std::endl;
+            return 1;
+        }
+        return runAddress(arguments[2], arguments[3]);
+    }
     std::cerr << "usage: ex04_type_boundary MODE ..." << std::endl;
     return 1;
 }


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
 
