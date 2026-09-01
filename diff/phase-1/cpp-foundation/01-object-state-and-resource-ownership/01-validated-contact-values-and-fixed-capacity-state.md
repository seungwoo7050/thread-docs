# 검증된 연락처 값과 고정 용량 상태

## `feat(contact): 검증된 연락처 값 객체 구현`

diff --git a/Makefile b/Makefile
index e322486..6786a2f 100644
--- a/Makefile
+++ b/Makefile
@@ -22,11 +22,7 @@ all: $(NAME)
 
 $(NAME): $(OBJ)
 	$(RM) $@
-	@if test -n "$(strip $(OBJ))"; then \
-		$(AR) $(ARFLAGS) $@ $(OBJ); \
-	else \
-		printf '!<arch>\n' > $@; \
-	fi
+	$(AR) $(ARFLAGS) $@ $(OBJ)
 
 build/obj/%.o: src/%.cpp
 	@$(MKDIR) $(dir $@)
diff --git a/include/cppf/Contact.hpp b/include/cppf/Contact.hpp
new file mode 100644
index 0000000..5800109
--- /dev/null
+++ b/include/cppf/Contact.hpp
@@ -0,0 +1,27 @@
+#ifndef CPPF_CONTACT_HPP
+#define CPPF_CONTACT_HPP
+
+#include <string>
+
+namespace cppf
+{
+
+class Contact
+{
+public:
+    Contact();
+    Contact(const std::string &name, const std::string &note);
+
+    bool empty() const;
+    const std::string &name() const;
+    const std::string &note() const;
+    void swap(Contact &other) throw();
+
+private:
+    std::string name_;
+    std::string note_;
+};
+
+}
+
+#endif
diff --git a/src/Contact.cpp b/src/Contact.cpp
new file mode 100644
index 0000000..cc94817
--- /dev/null
+++ b/src/Contact.cpp
@@ -0,0 +1,61 @@
+#include "cppf/Contact.hpp"
+
+namespace
+{
+
+bool validField(const std::string &value, std::size_t limit, bool allow_empty)
+{
+    std::string::size_type index;
+
+    if ((!allow_empty && value.empty()) || value.size() > limit)
+        return false;
+    for (index = 0; index < value.size(); ++index)
+    {
+        const unsigned char byte = static_cast<unsigned char>(value[index]);
+        if (byte < 32 || byte > 126)
+            return false;
+    }
+    return true;
+}
+
+}
+
+namespace cppf
+{
+
+Contact::Contact() : name_(), note_()
+{
+}
+
+Contact::Contact(const std::string &name, const std::string &note)
+    : name_(), note_()
+{
+    if (validField(name, 32, false) && validField(note, 64, true))
+    {
+        name_ = name;
+        note_ = note;
+    }
+}
+
+bool Contact::empty() const
+{
+    return name_.empty();
+}
+
+const std::string &Contact::name() const
+{
+    return name_;
+}
+
+const std::string &Contact::note() const
+{
+    return note_;
+}
+
+void Contact::swap(Contact &other) throw()
+{
+    name_.swap(other.name_);
+    note_.swap(other.note_);
+}
+
+}


## `test(contact): 연락처 값 불변식 검증`

diff --git a/Makefile b/Makefile
index 6786a2f..2924754 100644
--- a/Makefile
+++ b/Makefile
@@ -4,7 +4,7 @@ CXX := c++
 override CXXFLAGS := -Wall -Wextra -Werror -Wpedantic -pedantic-errors \
 	-std=c++98 -Wold-style-cast -Wcast-qual -Woverloaded-virtual \
 	-Wnon-virtual-dtor -Wc++11-extensions
-override CPPFLAGS := -Iinclude
+override CPPFLAGS := -Iinclude -Itests
 DEPFLAGS := -MMD -MP
 AR := ar
 ARFLAGS := rcs
@@ -16,7 +16,10 @@ SRC := $(sort $(wildcard src/*.cpp))
 OBJ := $(SRC:src/%.cpp=build/obj/%.o)
 DEP := $(OBJ:.o=.d)
 
-.PHONY: all clean fclean re
+TEST_SRC := $(sort $(wildcard tests/test_*.cpp))
+TEST_BIN := build/tests/unit
+
+.PHONY: all test check clean fclean re
 
 all: $(NAME)
 
@@ -28,6 +31,20 @@ build/obj/%.o: src/%.cpp
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(DEPFLAGS) -c $< -o $@
 
+$(TEST_BIN): $(TEST_SRC) $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(TEST_SRC) $(NAME) -o $@
+
+test: $(TEST_BIN)
+	./$(TEST_BIN)
+
+check:
+	git diff --check
+	$(MAKE) fclean
+	$(MAKE) all
+	$(MAKE) test
+	$(MAKE) -q all
+
 clean:
 	$(RMDIR) build
 
diff --git a/tests/support/Test.hpp b/tests/support/Test.hpp
new file mode 100644
index 0000000..3ed5d64
--- /dev/null
+++ b/tests/support/Test.hpp
@@ -0,0 +1,46 @@
+#ifndef CPP_FOUNDATION_TEST_HPP
+#define CPP_FOUNDATION_TEST_HPP
+
+#include <iostream>
+
+namespace test_support
+{
+
+class Suite
+{
+public:
+    Suite() : checks_(0), failures_(0)
+    {
+    }
+
+    void check(bool condition, const char *label)
+    {
+        ++checks_;
+        if (!condition)
+        {
+            ++failures_;
+            std::cerr << "FAIL: " << label << std::endl;
+        }
+    }
+
+    unsigned int checks() const
+    {
+        return checks_;
+    }
+
+    int result() const
+    {
+        if (failures_ != 0)
+            return 1;
+        std::cout << checks_ << " checks passed" << std::endl;
+        return 0;
+    }
+
+private:
+    unsigned int checks_;
+    unsigned int failures_;
+};
+
+}
+
+#endif
diff --git a/tests/test_contact.cpp b/tests/test_contact.cpp
new file mode 100644
index 0000000..f37c6f0
--- /dev/null
+++ b/tests/test_contact.cpp
@@ -0,0 +1,31 @@
+#include "cppf/Contact.hpp"
+#include "support/Test.hpp"
+
+#include <string>
+
+void testContact(test_support::Suite &suite)
+{
+    cppf::Contact empty;
+    cppf::Contact contact("Ada", "analytical engine");
+    const cppf::Contact &view = contact;
+    cppf::Contact copy(contact);
+    cppf::Contact other("Grace", "compiler");
+
+    suite.check(empty.empty(), "contact default is empty");
+    suite.check(!contact.empty(), "contact value is present");
+    suite.check(view.name() == "Ada", "contact const name accessor");
+    suite.check(view.note() == "analytical engine", "contact const note accessor");
+    suite.check(copy.name() == contact.name(), "contact value copy name");
+    suite.check(copy.note() == contact.note(), "contact value copy note");
+
+    contact.swap(other);
+    suite.check(contact.name() == "Grace", "contact swap destination");
+    suite.check(other.name() == "Ada", "contact swap source");
+    suite.check(cppf::Contact("", "note").empty(), "contact rejects empty name");
+    suite.check(cppf::Contact(std::string(33, 'x'), "note").empty(),
+                "contact rejects long name");
+    suite.check(cppf::Contact("name", std::string(65, 'x')).empty(),
+                "contact rejects long note");
+    suite.check(cppf::Contact("bad\nname", "note").empty(),
+                "contact rejects control bytes");
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
new file mode 100644
index 0000000..6cf1397
--- /dev/null
+++ b/tests/test_main.cpp
@@ -0,0 +1,11 @@
+#include "support/Test.hpp"
+
+void testContact(test_support::Suite &suite);
+
+int main()
+{
+    test_support::Suite suite;
+
+    testContact(suite);
+    return suite.result();
+}


## `feat(contact): 고정 크기 연락처 저장 순서 보존`

diff --git a/include/cppf/ContactBook.hpp b/include/cppf/ContactBook.hpp
new file mode 100644
index 0000000..cc55beb
--- /dev/null
+++ b/include/cppf/ContactBook.hpp
@@ -0,0 +1,35 @@
+#ifndef CPPF_CONTACT_BOOK_HPP
+#define CPPF_CONTACT_BOOK_HPP
+
+#include "cppf/Contact.hpp"
+
+#include <cstddef>
+#include <iosfwd>
+
+namespace cppf
+{
+
+class ContactBook
+{
+public:
+    enum
+    {
+        capacity = 8
+    };
+
+    ContactBook();
+
+    void add(const Contact &contact);
+    std::size_t size() const;
+    const Contact &at(std::size_t logical_index) const;
+    void write(std::ostream &output) const;
+
+private:
+    Contact contacts_[capacity];
+    std::size_t size_;
+    std::size_t next_;
+};
+
+}
+
+#endif
diff --git a/src/ContactBook.cpp b/src/ContactBook.cpp
new file mode 100644
index 0000000..974892b
--- /dev/null
+++ b/src/ContactBook.cpp
@@ -0,0 +1,37 @@
+#include "cppf/ContactBook.hpp"
+
+#include <stdexcept>
+
+namespace cppf
+{
+
+ContactBook::ContactBook() : contacts_(), size_(0), next_(0)
+{
+}
+
+void ContactBook::add(const Contact &contact)
+{
+    if (contact.empty())
+        return;
+    contacts_[next_] = contact;
+    next_ = (next_ + 1) % capacity;
+    if (size_ < capacity)
+        ++size_;
+}
+
+std::size_t ContactBook::size() const
+{
+    return size_;
+}
+
+const Contact &ContactBook::at(std::size_t logical_index) const
+{
+    std::size_t first;
+
+    if (logical_index >= size_)
+        throw std::out_of_range("contact index");
+    first = size_ == capacity ? next_ : 0;
+    return contacts_[(first + logical_index) % capacity];
+}
+
+}


## `test(contact): 연락처 저장 용량과 논리 순서 검증`

diff --git a/tests/test_contact_book.cpp b/tests/test_contact_book.cpp
new file mode 100644
index 0000000..02d80ec
--- /dev/null
+++ b/tests/test_contact_book.cpp
@@ -0,0 +1,36 @@
+#include "cppf/ContactBook.hpp"
+#include "support/Test.hpp"
+
+#include <stdexcept>
+
+void testContactBook(test_support::Suite &suite)
+{
+    cppf::ContactBook book;
+    const char *names[] = {
+        "A", "B", "C", "D", "E", "F", "G", "H", "I", "J"
+    };
+    std::size_t index;
+    bool threw;
+
+    suite.check(book.size() == 0, "contact book starts empty");
+    book.add(cppf::Contact());
+    suite.check(book.size() == 0, "contact book ignores empty values");
+    for (index = 0; index < 10; ++index)
+        book.add(cppf::Contact(names[index], "note"));
+    suite.check(book.size() == cppf::ContactBook::capacity,
+                "contact book keeps bounded size");
+    suite.check(book.at(0).name() == "C", "contact book replaces oldest");
+    suite.check(book.at(7).name() == "J", "contact book keeps newest");
+    suite.check(book.at(3).name() == "F", "contact book maps logical order");
+
+    threw = false;
+    try
+    {
+        book.at(book.size());
+    }
+    catch (const std::out_of_range &)
+    {
+        threw = true;
+    }
+    suite.check(threw, "contact book rejects invalid index");
+}
diff --git a/tests/test_main.cpp b/tests/test_main.cpp
index 6cf1397..326d7d0 100644
--- a/tests/test_main.cpp
+++ b/tests/test_main.cpp
@@ -1,11 +1,13 @@
 #include "support/Test.hpp"
 
 void testContact(test_support::Suite &suite);
+void testContactBook(test_support::Suite &suite);
 
 int main()
 {
     test_support::Suite suite;
 
     testContact(suite);
+    testContactBook(suite);
     return suite.result();
 }


## `feat(contact): 연락처 목록의 스트림 출력을 지원`

diff --git a/src/ContactBook.cpp b/src/ContactBook.cpp
index 974892b..08a9cc3 100644
--- a/src/ContactBook.cpp
+++ b/src/ContactBook.cpp
@@ -1,5 +1,6 @@
 #include "cppf/ContactBook.hpp"
 
+#include <ostream>
 #include <stdexcept>
 
 namespace cppf
@@ -34,4 +35,16 @@ const Contact &ContactBook::at(std::size_t logical_index) const
     return contacts_[(first + logical_index) % capacity];
 }
 
+void ContactBook::write(std::ostream &output) const
+{
+    std::size_t index;
+
+    for (index = 0; index < size_; ++index)
+    {
+        const Contact &contact = at(index);
+        output << index << '|' << contact.name() << '|' << contact.note()
+               << '\n';
+    }
+}
+
 }


## `feat(contact): 연락처 명령행 세션 연결`

diff --git a/Makefile b/Makefile
index 2924754..8d03209 100644
--- a/Makefile
+++ b/Makefile
@@ -16,12 +16,15 @@ SRC := $(sort $(wildcard src/*.cpp))
 OBJ := $(SRC:src/%.cpp=build/obj/%.o)
 DEP := $(OBJ:.o=.d)
 
+APP_SRC := $(sort $(wildcard apps/*.cpp))
+APP_BIN := $(APP_SRC:apps/%.cpp=bin/%)
+
 TEST_SRC := $(sort $(wildcard tests/test_*.cpp))
 TEST_BIN := build/tests/unit
 
 .PHONY: all test check clean fclean re
 
-all: $(NAME)
+all: $(NAME) $(APP_BIN)
 
 $(NAME): $(OBJ)
 	$(RM) $@
@@ -31,6 +34,10 @@ build/obj/%.o: src/%.cpp
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(DEPFLAGS) -c $< -o $@
 
+bin/%: apps/%.cpp $(NAME)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $< $(NAME) -o $@
+
 $(TEST_BIN): $(TEST_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(TEST_SRC) $(NAME) -o $@
@@ -46,7 +53,7 @@ check:
 	$(MAKE) -q all
 
 clean:
-	$(RMDIR) build
+	$(RMDIR) build bin
 
 fclean: clean
 	$(RM) $(NAME)
diff --git a/apps/ex00_contact_book.cpp b/apps/ex00_contact_book.cpp
new file mode 100644
index 0000000..74a042e
--- /dev/null
+++ b/apps/ex00_contact_book.cpp
@@ -0,0 +1,44 @@
+#include "cppf/Contact.hpp"
+#include "cppf/ContactBook.hpp"
+
+#include <iostream>
+#include <string>
+
+namespace
+{
+
+bool addContact(cppf::ContactBook &book, const std::string &payload)
+{
+    const std::string::size_type separator = payload.find('|');
+    cppf::Contact contact;
+
+    if (separator == std::string::npos)
+        return false;
+    contact = cppf::Contact(payload.substr(0, separator),
+                            payload.substr(separator + 1));
+    if (contact.empty())
+        return false;
+    book.add(contact);
+    return true;
+}
+
+}
+
+int main()
+{
+    cppf::ContactBook book;
+    std::string line;
+
+    while (std::getline(std::cin, line))
+    {
+        if (line.compare(0, 4, "ADD ") == 0)
+            std::cout << (addContact(book, line.substr(4)) ? "ok\n" : "error\n");
+        else if (line == "LIST")
+            book.write(std::cout);
+        else if (line == "QUIT")
+            return 0;
+        else
+            std::cout << "error\n";
+    }
+    return 0;
+}


## `test(contact): 공개 계약과 명령행 세션 검증`

diff --git a/Makefile b/Makefile
index 8d03209..36fae6a 100644
--- a/Makefile
+++ b/Makefile
@@ -22,7 +22,7 @@ APP_BIN := $(APP_SRC:apps/%.cpp=bin/%)
 TEST_SRC := $(sort $(wildcard tests/test_*.cpp))
 TEST_BIN := build/tests/unit
 
-.PHONY: all test check clean fclean re
+.PHONY: all test-unit test-contract test-integration test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -42,9 +42,20 @@ $(TEST_BIN): $(TEST_SRC) $(NAME)
 	@$(MKDIR) $(dir $@)
 	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(TEST_SRC) $(NAME) -o $@
 
-test: $(TEST_BIN)
+test-unit: $(TEST_BIN)
 	./$(TEST_BIN)
 
+test-contract:
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/contact_headers.cpp
+	@! $(CXX) $(CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
+		tests/compile/contact_private_fail.cpp >/dev/null 2>&1
+
+test-integration: bin/ex00_contact_book
+	sh tests/check_cli.sh
+
+test: test-unit test-contract test-integration
+
 check:
 	git diff --check
 	$(MAKE) fclean
diff --git a/tests/check_cli.sh b/tests/check_cli.sh
new file mode 100644
index 0000000..7fcc8c8
--- /dev/null
+++ b/tests/check_cli.sh
@@ -0,0 +1,10 @@
+#!/bin/sh
+
+set -eu
+
+temporary_directory=$(mktemp -d "${TMPDIR:-/tmp}/cpp-foundation-cli.XXXXXX")
+trap 'rm -rf "$temporary_directory"' EXIT HUP INT TERM
+
+./bin/ex00_contact_book < tests/fixtures/contact-session.in \
+    > "$temporary_directory/contact.out"
+diff -u tests/fixtures/contact-session.out "$temporary_directory/contact.out"
diff --git a/tests/compile/contact_headers.cpp b/tests/compile/contact_headers.cpp
new file mode 100644
index 0000000..50bee40
--- /dev/null
+++ b/tests/compile/contact_headers.cpp
@@ -0,0 +1,17 @@
+#include "cppf/Contact.hpp"
+#include "cppf/Contact.hpp"
+#include "cppf/ContactBook.hpp"
+#include "cppf/ContactBook.hpp"
+
+#include <sstream>
+
+int main()
+{
+    cppf::ContactBook book;
+    const cppf::Contact contact("Ada", "math");
+    std::ostringstream output;
+
+    book.add(contact);
+    book.write(output);
+    return book.at(0).name() == "Ada" ? 0 : 1;
+}
diff --git a/tests/compile/contact_private_fail.cpp b/tests/compile/contact_private_fail.cpp
new file mode 100644
index 0000000..2ea8f11
--- /dev/null
+++ b/tests/compile/contact_private_fail.cpp
@@ -0,0 +1,8 @@
+#include "cppf/Contact.hpp"
+
+int main()
+{
+    cppf::Contact contact;
+
+    return contact.name_.empty() ? 0 : 1;
+}
diff --git a/tests/fixtures/contact-session.in b/tests/fixtures/contact-session.in
new file mode 100644
index 0000000..c106538
--- /dev/null
+++ b/tests/fixtures/contact-session.in
@@ -0,0 +1,5 @@
+ADD Ada|math
+ADD Grace|compiler
+BAD
+LIST
+QUIT
diff --git a/tests/fixtures/contact-session.out b/tests/fixtures/contact-session.out
new file mode 100644
index 0000000..de6c0f4
--- /dev/null
+++ b/tests/fixtures/contact-session.out
@@ -0,0 +1,5 @@
+ok
+ok
+error
+0|Ada|math
+1|Grace|compiler
diff --git a/tests/test_contact_book.cpp b/tests/test_contact_book.cpp
index 02d80ec..192632d 100644
--- a/tests/test_contact_book.cpp
+++ b/tests/test_contact_book.cpp
@@ -1,6 +1,7 @@
 #include "cppf/ContactBook.hpp"
 #include "support/Test.hpp"
 
+#include <sstream>
 #include <stdexcept>
 
 void testContactBook(test_support::Suite &suite)
@@ -11,6 +12,7 @@ void testContactBook(test_support::Suite &suite)
     };
     std::size_t index;
     bool threw;
+    std::ostringstream output;
 
     suite.check(book.size() == 0, "contact book starts empty");
     book.add(cppf::Contact());
@@ -22,6 +24,13 @@ void testContactBook(test_support::Suite &suite)
     suite.check(book.at(0).name() == "C", "contact book replaces oldest");
     suite.check(book.at(7).name() == "J", "contact book keeps newest");
     suite.check(book.at(3).name() == "F", "contact book maps logical order");
+    suite.check(book.at(0).name() != "A", "contact book discards first value");
+    suite.check(book.at(0).name() != "B", "contact book discards second value");
+    book.write(output);
+    suite.check(output.str() ==
+                    "0|C|note\n1|D|note\n2|E|note\n3|F|note\n"
+                    "4|G|note\n5|H|note\n6|I|note\n7|J|note\n",
+                "contact book writes logical order");
 
     threw = false;
     try


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
