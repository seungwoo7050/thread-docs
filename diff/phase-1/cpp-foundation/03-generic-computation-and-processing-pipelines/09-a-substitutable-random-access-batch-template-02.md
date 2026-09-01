## `test(contracts): 공개 include와 소유권 규칙 검증`

diff --git a/Makefile b/Makefile
index b7dd496..8feb601 100644
--- a/Makefile
+++ b/Makefile
@@ -5,6 +5,7 @@ override CXXFLAGS := -Wall -Wextra -Werror -Wpedantic -pedantic-errors \
 	-std=c++98 -Wold-style-cast -Wcast-qual -Woverloaded-virtual \
 	-Wnon-virtual-dtor -Wc++11-extensions
 override CPPFLAGS := -Iinclude -Itests
+PUBLIC_CPPFLAGS := -Iinclude
 DEPFLAGS := -MMD -MP
 AR := ar
 ARFLAGS := rcs
@@ -32,6 +33,8 @@ BATCH_FAILURE_BIN := build/tests/batch_failure
 BATCH_FAILURE_SRC := tests/failure/test_batch_failure.cpp \
 	tests/support/FailingNew.cpp
 NO_ELIDE_BIN := build/tests/unit_no_elide
+PUBLIC_CONTRACT_BIN := build/tests/public_contract
+PUBLIC_CONTRACT_SRC := tests/integration/test_public_contract.cpp
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
 	test-integration test check clean fclean re
@@ -83,42 +86,74 @@ $(NO_ELIDE_BIN): $(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME)
 test-no-elide: $(NO_ELIDE_BIN)
 	./$(NO_ELIDE_BIN)
 
+$(PUBLIC_CONTRACT_BIN): $(PUBLIC_CONTRACT_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) $(PUBLIC_CONTRACT_SRC) \
+		$(NAME) -o $@
+
 test-contract:
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/public_headers.cpp
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/text_buffer_headers.cpp
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/format_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/factory_headers.cpp
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/scalar_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/runtime_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/serializer_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/template_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/rpn_headers.cpp
-	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	$(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/batch_headers.cpp
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/formatter_abstract_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/runtime_inspector_private_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/runtime_unrelated_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/serializer_private_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/serializer_const_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/rpn_evaluator_private_fail.cpp >/dev/null 2>&1
-	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/batch_results_mutation_fail.cpp >/dev/null 2>&1
-
-test-integration: $(APP_BIN)
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/contact_book_const_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/text_buffer_const_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/text_buffer_storage_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/text_buffer_implicit_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/formatter_creator_abstract_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/pipeline_builder_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/scalar_converter_private_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/runtime_base_constructor_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/template_const_iterator_fail.cpp >/dev/null 2>&1
+	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/template_list_sort_fail.cpp >/dev/null 2>&1
+
+test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 	sh tests/check_cli.sh
+	./$(PUBLIC_CONTRACT_BIN)
 
 test: test-unit failure-test test-no-elide test-contract test-integration
 
diff --git a/tests/compile/contact_book_const_fail.cpp b/tests/compile/contact_book_const_fail.cpp
new file mode 100644
index 0000000..07c8750
--- /dev/null
+++ b/tests/compile/contact_book_const_fail.cpp
@@ -0,0 +1,11 @@
+#include "cppf/ContactBook.hpp"
+
+int main()
+{
+    cppf::ContactBook book;
+    cppf::Contact replacement("Grace", "ownership");
+
+    book.add(cppf::Contact("Ada", "objects"));
+    book.at(0).swap(replacement);
+    return 0;
+}
diff --git a/tests/compile/factory_headers.cpp b/tests/compile/factory_headers.cpp
new file mode 100644
index 0000000..c3e6384
--- /dev/null
+++ b/tests/compile/factory_headers.cpp
@@ -0,0 +1,14 @@
+#include "cppf/Factory.hpp"
+#include "cppf/Factory.hpp"
+
+int main()
+{
+    const cppf::DefaultFormatterCreator creator;
+    cppf::Formatter *formatter = creator.create("upper");
+    const std::string specifications[] = {"prefix=[", "suffix=]"};
+    cppf::FormatPipeline pipeline;
+
+    delete formatter;
+    cppf::PipelineBuilder::replace(pipeline, creator, specifications, 2);
+    return pipeline.size() != 2;
+}
diff --git a/tests/compile/formatter_creator_abstract_fail.cpp b/tests/compile/formatter_creator_abstract_fail.cpp
new file mode 100644
index 0000000..0b1a4e3
--- /dev/null
+++ b/tests/compile/formatter_creator_abstract_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/Factory.hpp"
+
+int main()
+{
+    cppf::FormatterCreator creator;
+    return 0;
+}
diff --git a/tests/compile/pipeline_builder_private_fail.cpp b/tests/compile/pipeline_builder_private_fail.cpp
new file mode 100644
index 0000000..bce3a3d
--- /dev/null
+++ b/tests/compile/pipeline_builder_private_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/Factory.hpp"
+
+int main()
+{
+    cppf::PipelineBuilder builder;
+    return 0;
+}
diff --git a/tests/compile/public_headers.cpp b/tests/compile/public_headers.cpp
new file mode 100644
index 0000000..6d15e99
--- /dev/null
+++ b/tests/compile/public_headers.cpp
@@ -0,0 +1,29 @@
+#include "cppf/BatchEngine.hpp"
+#include "cppf/BatchEngine.hpp"
+#include "cppf/Contact.hpp"
+#include "cppf/Contact.hpp"
+#include "cppf/ContactBook.hpp"
+#include "cppf/ContactBook.hpp"
+#include "cppf/Factory.hpp"
+#include "cppf/Factory.hpp"
+#include "cppf/FormatPipeline.hpp"
+#include "cppf/FormatPipeline.hpp"
+#include "cppf/Formatter.hpp"
+#include "cppf/Formatter.hpp"
+#include "cppf/RandomAccessBatch.hpp"
+#include "cppf/RandomAccessBatch.hpp"
+#include "cppf/RpnEvaluator.hpp"
+#include "cppf/RpnEvaluator.hpp"
+#include "cppf/RuntimeType.hpp"
+#include "cppf/RuntimeType.hpp"
+#include "cppf/ScalarConverter.hpp"
+#include "cppf/ScalarConverter.hpp"
+#include "cppf/Serializer.hpp"
+#include "cppf/Serializer.hpp"
+#include "cppf/TextBuffer.hpp"
+#include "cppf/TextBuffer.hpp"
+
+int main()
+{
+    return 0;
+}
diff --git a/tests/compile/runtime_base_constructor_fail.cpp b/tests/compile/runtime_base_constructor_fail.cpp
new file mode 100644
index 0000000..6cdd73a
--- /dev/null
+++ b/tests/compile/runtime_base_constructor_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/RuntimeType.hpp"
+
+int main()
+{
+    cppf::RuntimeBase value;
+    return 0;
+}
diff --git a/tests/compile/scalar_converter_private_fail.cpp b/tests/compile/scalar_converter_private_fail.cpp
new file mode 100644
index 0000000..dba997e
--- /dev/null
+++ b/tests/compile/scalar_converter_private_fail.cpp
@@ -0,0 +1,7 @@
+#include "cppf/ScalarConverter.hpp"
+
+int main()
+{
+    cppf::ScalarConverter converter;
+    return 0;
+}
diff --git a/tests/compile/template_const_iterator_fail.cpp b/tests/compile/template_const_iterator_fail.cpp
new file mode 100644
index 0000000..112b0a8
--- /dev/null
+++ b/tests/compile/template_const_iterator_fail.cpp
@@ -0,0 +1,9 @@
+#include "cppf/RandomAccessBatch.hpp"
+
+int main()
+{
+    const cppf::RandomAccessBatch<int> values;
+
+    *values.begin() = 1;
+    return 0;
+}
diff --git a/tests/compile/template_headers.cpp b/tests/compile/template_headers.cpp
index d9a6c27..d10da03 100644
--- a/tests/compile/template_headers.cpp
+++ b/tests/compile/template_headers.cpp
@@ -10,10 +10,18 @@ bool lessInt(int left, int right)
 
 int main()
 {
-    cppf::RandomAccessBatch<int, std::deque<int> > values;
+    cppf::RandomAccessBatch<int> vector_values;
+    cppf::RandomAccessBatch<int, std::deque<int> > deque_values;
 
-    values.push_back(2);
-    values.push_back(1);
-    values.sort(lessInt);
-    return values.at(0) != 1;
+    vector_values.push_back(2);
+    vector_values.push_back(1);
+    deque_values.push_back(2);
+    deque_values.push_back(1);
+    vector_values.sort(lessInt);
+    deque_values.sort(lessInt);
+    const cppf::RandomAccessBatch<int> &vector_view = vector_values;
+    const cppf::RandomAccessBatch<int, std::deque<int> > &deque_view =
+        deque_values;
+    return !cppf::equal_ranges(vector_view.begin(), vector_view.end(),
+                               deque_view.begin(), deque_view.end());
 }
diff --git a/tests/compile/template_list_sort_fail.cpp b/tests/compile/template_list_sort_fail.cpp
new file mode 100644
index 0000000..fba3501
--- /dev/null
+++ b/tests/compile/template_list_sort_fail.cpp
@@ -0,0 +1,18 @@
+#include "cppf/RandomAccessBatch.hpp"
+
+#include <list>
+
+bool lessInt(int left, int right)
+{
+    return left < right;
+}
+
+int main()
+{
+    cppf::RandomAccessBatch<int, std::list<int> > values;
+
+    values.push_back(2);
+    values.push_back(1);
+    values.sort(lessInt);
+    return 0;
+}
diff --git a/tests/compile/text_buffer_const_fail.cpp b/tests/compile/text_buffer_const_fail.cpp
new file mode 100644
index 0000000..a6b6dea
--- /dev/null
+++ b/tests/compile/text_buffer_const_fail.cpp
@@ -0,0 +1,9 @@
+#include "cppf/TextBuffer.hpp"
+
+int main()
+{
+    const cppf::TextBuffer value("fixed");
+
+    value.at(0) = 'F';
+    return 0;
+}
diff --git a/tests/compile/text_buffer_headers.cpp b/tests/compile/text_buffer_headers.cpp
new file mode 100644
index 0000000..3043356
--- /dev/null
+++ b/tests/compile/text_buffer_headers.cpp
@@ -0,0 +1,11 @@
+#include "cppf/TextBuffer.hpp"
+#include "cppf/TextBuffer.hpp"
+
+int main()
+{
+    cppf::TextBuffer value("value");
+    const cppf::TextBuffer &view = value;
+
+    value.at(0) = 'V';
+    return view.c_str()[0] != 'V';
+}
diff --git a/tests/compile/text_buffer_implicit_fail.cpp b/tests/compile/text_buffer_implicit_fail.cpp
new file mode 100644
index 0000000..51c94be
--- /dev/null
+++ b/tests/compile/text_buffer_implicit_fail.cpp
@@ -0,0 +1,12 @@
+#include "cppf/TextBuffer.hpp"
+
+void consume(const cppf::TextBuffer &value)
+{
+    (void)value;
+}
+
+int main()
+{
+    consume("implicit");
+    return 0;
+}
diff --git a/tests/compile/text_buffer_storage_fail.cpp b/tests/compile/text_buffer_storage_fail.cpp
new file mode 100644
index 0000000..89365cf
--- /dev/null
+++ b/tests/compile/text_buffer_storage_fail.cpp
@@ -0,0 +1,9 @@
+#include "cppf/TextBuffer.hpp"
+
+int main()
+{
+    cppf::TextBuffer value("fixed");
+
+    value.c_str()[0] = 'F';
+    return 0;
+}
diff --git a/tests/integration/test_public_contract.cpp b/tests/integration/test_public_contract.cpp
new file mode 100644
index 0000000..7ddca0c
--- /dev/null
+++ b/tests/integration/test_public_contract.cpp
@@ -0,0 +1,123 @@
+#include "cppf/BatchEngine.hpp"
+#include "cppf/Contact.hpp"
+#include "cppf/ContactBook.hpp"
+#include "cppf/Factory.hpp"
+#include "cppf/FormatPipeline.hpp"
+#include "cppf/Formatter.hpp"
+#include "cppf/RandomAccessBatch.hpp"
+#include "cppf/RpnEvaluator.hpp"
+#include "cppf/RuntimeType.hpp"
+#include "cppf/ScalarConverter.hpp"
+#include "cppf/Serializer.hpp"
+#include "cppf/TextBuffer.hpp"
+
+#include <deque>
+#include <sstream>
+#include <string>
+
+namespace
+{
+
+bool resultLess(const cppf::JobResult &left,
+                const cppf::JobResult &right)
+{
+    if (left.value() != right.value())
+        return left.value() < right.value();
+    return left.name() < right.name();
+}
+
+}
+
+int main()
+{
+    cppf::ContactBook source;
+
+    source.add(cppf::Contact("alpha", "60 5 +"));
+    source.add(cppf::Contact("beta", "10 2 *"));
+    const cppf::ContactBook &records = source;
+    const cppf::TextBuffer owned_name(records.at(0).name().c_str());
+    cppf::TextBuffer independent_name(owned_name);
+    independent_name.at(0) = 'A';
+    if (owned_name != cppf::TextBuffer("alpha") ||
+        independent_name != cppf::TextBuffer("Alpha"))
+        return 1;
+
+    const cppf::DefaultFormatterCreator creator;
+    const std::string valid[] = {"prefix=[", "upper", "suffix=]"};
+    const std::string invalid[] = {"prefix=<", "unknown", "suffix=>"};
+    cppf::FormatPipeline pipeline;
+    bool rejected = false;
+
+    cppf::PipelineBuilder::replace(pipeline, creator, valid, 3);
+    const cppf::TextBuffer label = pipeline.apply(owned_name);
+    try
+    {
+        cppf::PipelineBuilder::replace(pipeline, creator, invalid, 3);
+    }
+    catch (const cppf::UnknownFormatter &)
+    {
+        rejected = true;
+    }
+    if (!rejected || label != cppf::TextBuffer("[ALPHA]") ||
+        pipeline.apply(owned_name) != label)
+        return 1;
+
+    std::ostringstream batch_input_text;
+    batch_input_text << records.at(0).name() << " | "
+                     << records.at(0).note() << '\n'
+                     << records.at(1).name() << " | "
+                     << records.at(1).note();
+    std::istringstream batch_input(batch_input_text.str());
+    cppf::BatchEngine engine;
+
+    engine.replace(batch_input);
+    if (engine.results().size() != 2 ||
+        !(engine.results()[0] == cppf::JobResult("beta", 20)) ||
+        !(engine.results()[1] == cppf::JobResult("alpha", 65)) ||
+        cppf::RpnEvaluator::evaluate(records.at(0).note()) != 65)
+        return 1;
+
+    cppf::RandomAccessBatch<cppf::JobResult> vector_results;
+    cppf::RandomAccessBatch<cppf::JobResult,
+                            std::deque<cppf::JobResult> > deque_results;
+    std::size_t index;
+
+    for (index = 0; index < engine.results().size(); ++index)
+    {
+        vector_results.push_back(engine.results()[index]);
+        deque_results.push_back(engine.results()[index]);
+    }
+    vector_results.sort(resultLess);
+    deque_results.sort(resultLess);
+    if (!cppf::equal_ranges(vector_results.begin(), vector_results.end(),
+                            deque_results.begin(), deque_results.end()))
+        return 1;
+
+    std::ostringstream scalar_output;
+    cppf::ScalarConverter::write("65", scalar_output);
+    if (scalar_output.str() !=
+        "char: 'A'\nint: 65\nfloat: 65.0f\ndouble: 65.0\n")
+        return 1;
+
+    cppf::RuntimeBase *runtime =
+        cppf::RuntimeInspector::create(cppf::runtime_a);
+    const bool runtime_matches =
+        runtime != 0 &&
+        cppf::RuntimeInspector::identify(runtime) == cppf::runtime_a &&
+        cppf::RuntimeInspector::identify(*runtime) == cppf::runtime_a;
+    delete runtime;
+    if (!runtime_matches)
+        return 1;
+
+    cppf::Payload payload(
+        static_cast<unsigned long>(engine.results()[1].value()),
+        label.c_str());
+    const cppf::Serializer::raw_type raw =
+        cppf::Serializer::serialize(&payload);
+    cppf::Payload *recovered = cppf::Serializer::deserialize(raw);
+
+    if (recovered != &payload || recovered->id != 65 ||
+        recovered->label != "[ALPHA]")
+        return 1;
+    return 0;
+}
