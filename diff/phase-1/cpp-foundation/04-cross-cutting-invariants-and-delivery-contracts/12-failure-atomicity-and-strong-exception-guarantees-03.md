## `fix(contact): 할당 실패에도 저장 상태 보존`

diff --git a/src/ContactBook.cpp b/src/ContactBook.cpp
index 08a9cc3..cf5767f 100644
--- a/src/ContactBook.cpp
+++ b/src/ContactBook.cpp
@@ -12,9 +12,12 @@ ContactBook::ContactBook() : contacts_(), size_(0), next_(0)
 
 void ContactBook::add(const Contact &contact)
 {
+    Contact replacement;
+
     if (contact.empty())
         return;
-    contacts_[next_] = contact;
+    replacement = contact;
+    contacts_[next_].swap(replacement);
     next_ = (next_ + 1) % capacity;
     if (size_ < capacity)
         ++size_;


## `test(contact): 연락처 교체 실패 회귀 검증`

diff --git a/Makefile b/Makefile
index df4118e..10035cd 100644
--- a/Makefile
+++ b/Makefile
@@ -35,6 +35,9 @@ BATCH_FAILURE_SRC := tests/failure/test_batch_failure.cpp \
 PIPELINE_FAILURE_BIN := build/tests/pipeline_failure
 PIPELINE_FAILURE_SRC := tests/failure/test_pipeline_failure.cpp \
 	tests/support/TestFormatter.cpp
+CONTACT_FAILURE_BIN := build/tests/contact_failure
+CONTACT_FAILURE_SRC := tests/failure/test_contact_failure.cpp \
+	tests/support/FailingNew.cpp
 NO_ELIDE_BIN := build/tests/unit_no_elide
 PUBLIC_CONTRACT_BIN := build/tests/public_contract
 PUBLIC_CONTRACT_SRC := tests/integration/test_public_contract.cpp
@@ -85,12 +88,17 @@ $(PIPELINE_FAILURE_BIN): $(PIPELINE_FAILURE_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(PIPELINE_FAILURE_SRC) $(NAME) -o $@
 
+$(CONTACT_FAILURE_BIN): $(CONTACT_FAILURE_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(CONTACT_FAILURE_SRC) $(NAME) -o $@
+
 failure-test: $(FAILURE_BIN) $(FACTORY_FAILURE_BIN) $(BATCH_FAILURE_BIN) \
-	$(PIPELINE_FAILURE_BIN)
+	$(PIPELINE_FAILURE_BIN) $(CONTACT_FAILURE_BIN)
 	./$(FAILURE_BIN)
 	./$(FACTORY_FAILURE_BIN)
 	./$(BATCH_FAILURE_BIN)
 	./$(PIPELINE_FAILURE_BIN)
+	./$(CONTACT_FAILURE_BIN)
 
 $(NO_ELIDE_BIN): $(TEST_SRC) $(TEST_SUPPORT_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
diff --git a/tests/failure/test_contact_failure.cpp b/tests/failure/test_contact_failure.cpp
new file mode 100644
index 0000000..9a212ac
--- /dev/null
+++ b/tests/failure/test_contact_failure.cpp
@@ -0,0 +1,117 @@
+#include "cppf/ContactBook.hpp"
+#include "support/FailingNew.hpp"
+
+#include <iostream>
+#include <new>
+#include <string>
+
+namespace
+{
+
+unsigned int checks = 0;
+unsigned int failures = 0;
+
+void check(bool condition)
+{
+    ++checks;
+    if (!condition)
+        ++failures;
+}
+
+void seed(cppf::ContactBook &book)
+{
+    const char *names[] = {
+        "one", "two", "three", "four", "five", "six", "seven", "eight"};
+    std::size_t index;
+
+    for (index = 0; index < cppf::ContactBook::capacity; ++index)
+        book.add(cppf::Contact(names[index], "seed"));
+}
+
+void checkSeed(const cppf::ContactBook &book)
+{
+    const char *names[] = {
+        "one", "two", "three", "four", "five", "six", "seven", "eight"};
+    std::size_t index;
+
+    check(book.size() == cppf::ContactBook::capacity);
+    for (index = 0; index < cppf::ContactBook::capacity; ++index)
+    {
+        check(book.at(index).name() == names[index]);
+        check(book.at(index).note() == "seed");
+    }
+}
+
+std::size_t successfulAddAllocationCount(const cppf::Contact &replacement)
+{
+    cppf::ContactBook book;
+
+    seed(book);
+    failing_new::resetAttempts();
+    book.add(replacement);
+    return failing_new::attempts();
+}
+
+void testAddFailureSweep()
+{
+    const std::string long_name(32, 'n');
+    const std::string long_note(64, 'x');
+    const cppf::Contact replacement(long_name, long_note);
+    const std::size_t allocation_count =
+        successfulAddAllocationCount(replacement);
+    const std::size_t outer_baseline = failing_new::liveBlocks();
+    std::size_t failure_index;
+
+    check(allocation_count >= 2);
+    for (failure_index = 1; failure_index <= allocation_count;
+         ++failure_index)
+    {
+        cppf::ContactBook book;
+        bool bad_allocation = false;
+        bool unexpected_exception = false;
+
+        seed(book);
+        const std::size_t baseline = failing_new::liveBlocks();
+        failing_new::failOn(failure_index);
+        try
+        {
+            book.add(replacement);
+        }
+        catch (const std::bad_alloc &)
+        {
+            bad_allocation = true;
+        }
+        catch (...)
+        {
+            unexpected_exception = true;
+        }
+        failing_new::disableFailure();
+
+        check(bad_allocation);
+        check(!unexpected_exception);
+        check(failing_new::attempts() == failure_index);
+        check(failing_new::liveBlocks() == baseline);
+        checkSeed(book);
+
+        book.add(replacement);
+        check(book.size() == cppf::ContactBook::capacity);
+        check(book.at(0).name() == "two");
+        check(book.at(cppf::ContactBook::capacity - 1).name() == long_name);
+        check(book.at(cppf::ContactBook::capacity - 1).note() == long_note);
+    }
+    check(failing_new::liveBlocks() == outer_baseline);
+}
+
+}
+
+int main()
+{
+    testAddFailureSweep();
+    if (failures != 0)
+    {
+        std::cerr << failures << " contact failure checks failed" << std::endl;
+        return 1;
+    }
+    std::cout << checks << " contact failure checks passed" << std::endl;
+    return 0;
+}
