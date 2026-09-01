## `test(template): iterator·정렬·복사 실패 계약 검증`

diff --git a/Makefile b/Makefile
index a29f156..36d96d3 100644
--- a/Makefile
+++ b/Makefile
@@ -86,6 +86,8 @@ test-contract:
 		tests/compile/runtime_headers.cpp
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/serializer_headers.cpp
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/template_headers.cpp
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
 	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
diff --git a/tests/compile/template_headers.cpp b/tests/compile/template_headers.cpp
new file mode 100644
index 0000000..d9a6c27
--- /dev/null
+++ b/tests/compile/template_headers.cpp
@@ -0,0 +1,19 @@
+#include "cppf/RandomAccessBatch.hpp"
+#include "cppf/RandomAccessBatch.hpp"
+
+#include <deque>
+
+bool lessInt(int left, int right)
+{
+    return left < right;
+}
+
+int main()
+{
+    cppf::RandomAccessBatch<int, std::deque<int> > values;
+
+    values.push_back(2);
+    values.push_back(1);
+    values.sort(lessInt);
+    return values.at(0) != 1;
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index c9686bc..921ae42 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -10,6 +10,7 @@ void testScalarLiteral(test_support::Suite &suite);
 void testScalarConverter(test_support::Suite &suite);
 void testRuntimeType(test_support::Suite &suite);
 void testSerializer(test_support::Suite &suite);
+void testRandomAccessBatch(test_support::Suite &suite);
 
 int main()
 {
@@ -25,5 +26,6 @@ int main()
     testScalarConverter(suite);
     testRuntimeType(suite);
     testSerializer(suite);
+    testRandomAccessBatch(suite);
     return suite.result();
 }
diff --git a/tests/test_random_access_batch.cpp b/tests/test_random_access_batch.cpp
new file mode 100644
index 0000000..3c9986f
--- /dev/null
+++ b/tests/test_random_access_batch.cpp
@@ -0,0 +1,283 @@
+#include "cppf/RandomAccessBatch.hpp"
+#include "support/Test.hpp"
+
+#include <algorithm>
+#include <deque>
+#include <iterator>
+#include <numeric>
+#include <stdexcept>
+#include <string>
+#include <vector>
+
+namespace
+{
+
+bool descending(int left, int right)
+{
+    return left > right;
+}
+
+class ValueCopyFailure
+{
+};
+
+class ThrowingValue
+{
+public:
+    explicit ThrowingValue(int value = 0) : value_(value)
+    {
+        ++live_count_;
+    }
+
+    ThrowingValue(const ThrowingValue &other) : value_(other.value_)
+    {
+        ++copy_attempts_;
+        if (failure_attempt_ != 0 && copy_attempts_ == failure_attempt_)
+            throw ValueCopyFailure();
+        ++live_count_;
+    }
+
+    ~ThrowingValue()
+    {
+        --live_count_;
+    }
+
+    ThrowingValue &operator=(const ThrowingValue &other)
+    {
+        value_ = other.value_;
+        return *this;
+    }
+
+    int value() const
+    {
+        return value_;
+    }
+
+    bool operator==(const ThrowingValue &other) const
+    {
+        return value_ == other.value_;
+    }
+
+    static void failCopyOn(int attempt)
+    {
+        copy_attempts_ = 0;
+        failure_attempt_ = attempt;
+    }
+
+    static void disableFailure()
+    {
+        failure_attempt_ = 0;
+    }
+
+    static int copyAttempts()
+    {
+        return copy_attempts_;
+    }
+
+    static int liveCount()
+    {
+        return live_count_;
+    }
+
+private:
+    int value_;
+    static int copy_attempts_;
+    static int failure_attempt_;
+    static int live_count_;
+};
+
+int ThrowingValue::copy_attempts_ = 0;
+int ThrowingValue::failure_attempt_ = 0;
+int ThrowingValue::live_count_ = 0;
+
+}
+
+void testRandomAccessBatch(test_support::Suite &suite)
+{
+    cppf::RandomAccessBatch<int> values;
+
+    suite.check(values.empty() && values.size() == 0,
+                "template batch starts empty");
+    values.push_back(2);
+    values.push_back(1);
+    values.push_back(3);
+    suite.check(values.size() == 3 && values.at(0) == 2,
+                "template batch appends indexed values");
+    values.at(0) = 4;
+    const cppf::RandomAccessBatch<int> &constant_values = values;
+    suite.check(constant_values.at(0) == 4,
+                "template batch exposes const indexed access");
+    suite.check(std::distance(values.begin(), values.end()) == 3,
+                "template batch exposes mutable iterator range");
+    suite.check(std::find(constant_values.begin(), constant_values.end(), 1) !=
+                    constant_values.end(),
+                "template batch const iterators support algorithms");
+    *values.begin() = 5;
+    suite.check(values.at(0) == 5,
+                "template batch mutable iterator updates values");
+    *values.begin() = 4;
+    suite.check(std::accumulate(constant_values.begin(),
+                                constant_values.end(), 0) == 8,
+                "template batch iterators support accumulation");
+
+    values.sort(descending);
+    suite.check(values.at(0) == 4 && values.at(1) == 3 &&
+                    values.at(2) == 1,
+                "template batch delegates comparator ordering");
+
+    bool out_of_range = false;
+    try
+    {
+        values.at(3);
+    }
+    catch (const std::out_of_range &error)
+    {
+        out_of_range = std::string(error.what()) == "batch index";
+    }
+    suite.check(out_of_range, "template batch preserves bounds exception");
+
+    cppf::RandomAccessBatch<int> copy(values);
+    copy.at(0) = 9;
+    suite.check(values.at(0) == 4 && copy.at(0) == 9,
+                "template batch copy owns independent values");
+    cppf::RandomAccessBatch<int> assigned;
+    assigned.push_back(8);
+    suite.check(&(assigned = values) == &assigned &&
+                    assigned.size() == values.size(),
+                "template batch assignment replaces values");
+    const cppf::RandomAccessBatch<int> &self_alias = assigned;
+    assigned = self_alias;
+    suite.check(assigned.at(0) == 4,
+                "template batch self assignment preserves values");
+
+    cppf::RandomAccessBatch<int, std::deque<int> > deque_values;
+    deque_values.push_back(1);
+    deque_values.push_back(4);
+    deque_values.push_back(3);
+    deque_values.sort(descending);
+    suite.check(cppf::equal_ranges(
+                    values.begin(), values.end(),
+                    deque_values.begin(), deque_values.end()),
+                "range equality crosses container types");
+    deque_values.push_back(0);
+    suite.check(!cppf::equal_ranges(
+                    values.begin(), values.end(),
+                    deque_values.begin(), deque_values.end()),
+                "range equality rejects length mismatch");
+    suite.check(!cppf::equal_ranges(
+                    values.begin(), values.end(),
+                    deque_values.begin(), deque_values.begin() + 2),
+                "range equality rejects shorter second range");
+    deque_values.at(0) = 5;
+    suite.check(!cppf::equal_ranges(
+                    values.begin(), values.end(),
+                    deque_values.begin(), deque_values.begin() + 3),
+                "range equality rejects value mismatch");
+
+    cppf::RandomAccessBatch<std::string> words;
+    words.push_back("beta");
+    words.push_back("alpha");
+    suite.check(words.at(0) == "beta" && words.at(1) == "alpha",
+                "template batch supports non-scalar value type");
+
+    cppf::RandomAccessBatch<int> swap_left;
+    cppf::RandomAccessBatch<int> swap_right;
+    swap_left.push_back(1);
+    swap_right.push_back(2);
+    swap_right.push_back(3);
+    swap_left.swap(swap_right);
+    suite.check(swap_left.size() == 2 && swap_left.at(0) == 2 &&
+                    swap_right.size() == 1 && swap_right.at(0) == 1,
+                "template batch swaps backing containers");
+
+    cppf::RandomAccessBatch<int> empty_left;
+    std::vector<int> empty_right;
+    suite.check(cppf::equal_ranges(
+                    empty_left.begin(), empty_left.end(),
+                    empty_right.begin(), empty_right.end()),
+                "range equality handles empty ranges without subtraction");
+
+    {
+        const ThrowingValue value(1);
+        cppf::RandomAccessBatch<ThrowingValue> throwing_batch;
+        bool threw = false;
+
+        ThrowingValue::failCopyOn(1);
+        try
+        {
+            throwing_batch.push_back(value);
+        }
+        catch (const ValueCopyFailure &)
+        {
+            threw = true;
+        }
+        ThrowingValue::disableFailure();
+        suite.check(threw && throwing_batch.empty() &&
+                        ThrowingValue::liveCount() == 1,
+                    "template push failure preserves empty batch");
+    }
+    suite.check(ThrowingValue::liveCount() == 0,
+                "template push failure leaves no live values");
+
+    {
+        cppf::RandomAccessBatch<ThrowingValue> source;
+        source.push_back(ThrowingValue(1));
+        source.push_back(ThrowingValue(2));
+        source.push_back(ThrowingValue(3));
+        const int source_baseline = ThrowingValue::liveCount();
+        bool copy_threw = false;
+
+        ThrowingValue::failCopyOn(2);
+        try
+        {
+            cppf::RandomAccessBatch<ThrowingValue> failed_copy(source);
+        }
+        catch (const ValueCopyFailure &)
+        {
+            copy_threw = true;
+        }
+        ThrowingValue::disableFailure();
+        suite.check(copy_threw && source.size() == 3 &&
+                        ThrowingValue::liveCount() == source_baseline,
+                    "template copy failure cleans partial values");
+
+        cppf::RandomAccessBatch<ThrowingValue> destination;
+        destination.push_back(ThrowingValue(9));
+        const int assignment_baseline = ThrowingValue::liveCount();
+        bool assignment_threw = false;
+
+        ThrowingValue::failCopyOn(2);
+        try
+        {
+            destination = source;
+        }
+        catch (const ValueCopyFailure &)
+        {
+            assignment_threw = true;
+        }
+        ThrowingValue::disableFailure();
+        suite.check(assignment_threw && destination.size() == 1 &&
+                        destination.at(0).value() == 9 &&
+                        ThrowingValue::liveCount() == assignment_baseline,
+                    "template assignment failure preserves destination");
+
+        const cppf::RandomAccessBatch<ThrowingValue> &destination_alias =
+            destination;
+        bool self_threw = false;
+        ThrowingValue::failCopyOn(1);
+        try
+        {
+            destination = destination_alias;
+        }
+        catch (...)
+        {
+            self_threw = true;
+        }
+        ThrowingValue::disableFailure();
+        suite.check(!self_threw && ThrowingValue::copyAttempts() == 0 &&
+                        destination.at(0).value() == 9,
+                    "template self assignment performs no value copy");
+    }
+    suite.check(ThrowingValue::liveCount() == 0,
+                "template copy failures restore live value count");
+}


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


