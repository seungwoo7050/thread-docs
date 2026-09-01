## `test(release): 정적 archive와 외부 dependency 검증`

diff --git a/Makefile b/Makefile
index 21cb3ec..392c210 100644
--- a/Makefile
+++ b/Makefile
@@ -35,9 +35,10 @@ BATCH_FAILURE_SRC := tests/failure/test_batch_failure.cpp \
 NO_ELIDE_BIN := build/tests/unit_no_elide
 PUBLIC_CONTRACT_BIN := build/tests/public_contract
 PUBLIC_CONTRACT_SRC := tests/integration/test_public_contract.cpp
+RELEASE_BIN := $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
-	test-integration test check clean fclean re
+	test-integration check-archive check-dependencies test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -157,6 +158,12 @@ test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 	sh tests/check_cli.sh
 	./$(PUBLIC_CONTRACT_BIN)
 
+check-archive: $(NAME)
+	sh tests/check_archive.sh $(NAME)
+
+check-dependencies: $(RELEASE_BIN)
+	sh tests/check_dependencies.sh $(RELEASE_BIN)
+
 test: test-unit failure-test test-no-elide test-contract test-integration
 
 check:
@@ -164,6 +171,8 @@ check:
 	$(MAKE) fclean
 	$(MAKE) all
 	$(MAKE) test
+	$(MAKE) check-archive
+	$(MAKE) check-dependencies
 	$(MAKE) -q all
 
 clean:
diff --git a/tests/check_archive.sh b/tests/check_archive.sh
new file mode 100644
index 0000000..97d57f2
--- /dev/null
+++ b/tests/check_archive.sh
@@ -0,0 +1,51 @@
+#!/bin/sh
+
+set -eu
+
+if [ "$#" -ne 1 ] || [ ! -f "$1" ]
+then
+    printf '검사할 정적 라이브러리가 필요합니다.\n' >&2
+    exit 2
+fi
+if [ "$(/usr/bin/uname -s)" != "Darwin" ]
+then
+    printf '정적 라이브러리 검사는 macOS가 필요합니다.\n' >&2
+    exit 2
+fi
+
+archive=$1
+temporary_directory=$(/usr/bin/mktemp -d \
+    "${TMPDIR:-/tmp}/cpp-foundation-archive.XXXXXX")
+
+cleanup()
+{
+    /bin/rm -f "$temporary_directory/members" \
+        "$temporary_directory/symbols"
+    /bin/rmdir "$temporary_directory"
+}
+
+trap cleanup EXIT HUP INT TERM
+
+/usr/bin/ar -t "$archive" |
+    /usr/bin/awk '$0 != "__.SYMDEF" && $0 != "__.SYMDEF SORTED"' \
+    > "$temporary_directory/members"
+if ! /usr/bin/diff -u tests/manifests/archive-members.manifest \
+    "$temporary_directory/members"
+then
+    printf 'archive 객체 구성이 manifest와 다릅니다.\n' >&2
+    exit 1
+fi
+
+LC_ALL=C
+export LC_ALL
+/usr/bin/nm -gUWj "$archive" |
+    /usr/bin/c++filt |
+    /usr/bin/awk 'NF && substr($0, length($0), 1) != ":"' |
+    /usr/bin/sed -E 's/std::__[[:alnum:]_]+::/std::/g' |
+    /usr/bin/sort -u > "$temporary_directory/symbols"
+if ! /usr/bin/diff -u tests/manifests/archive-symbols.manifest \
+    "$temporary_directory/symbols"
+then
+    printf 'archive 외부 심볼이 manifest와 다릅니다.\n' >&2
+    exit 1
+fi
diff --git a/tests/check_dependencies.sh b/tests/check_dependencies.sh
new file mode 100644
index 0000000..da858ce
--- /dev/null
+++ b/tests/check_dependencies.sh
@@ -0,0 +1,45 @@
+#!/bin/sh
+
+set -eu
+
+if [ "$#" -eq 0 ] || [ "$(/usr/bin/uname -s)" != "Darwin" ]
+then
+    printf 'Mach-O 의존성 검사는 macOS 실행 파일이 필요합니다.\n' >&2
+    exit 2
+fi
+
+temporary_directory=$(/usr/bin/mktemp -d \
+    "${TMPDIR:-/tmp}/cpp-foundation-dependencies.XXXXXX")
+
+cleanup()
+{
+    /bin/rm -f "$temporary_directory/libraries"
+    /bin/rmdir "$temporary_directory"
+}
+
+trap cleanup EXIT HUP INT TERM
+
+for binary in "$@"
+do
+    if [ ! -x "$binary" ]
+    then
+        printf '실행 파일을 찾을 수 없습니다: %s\n' "$binary" >&2
+        exit 2
+    fi
+    /usr/bin/otool -L "$binary" |
+        /usr/bin/awk 'NR > 1 { print $1 }' |
+        LC_ALL=C /usr/bin/sort -u > "$temporary_directory/libraries"
+    if ! /usr/bin/diff -u tests/manifests/macho-libraries.manifest \
+        "$temporary_directory/libraries"
+    then
+        printf '허용되지 않은 Mach-O 의존성: %s\n' "$binary" >&2
+        exit 1
+    fi
+    if /usr/bin/otool -l "$binary" |
+        /usr/bin/awk '$1 == "cmd" && $2 == "LC_RPATH" { found = 1 }
+            END { exit found ? 0 : 1 }'
+    then
+        printf 'LC_RPATH를 포함한 실행 파일: %s\n' "$binary" >&2
+        exit 1
+    fi
+done
diff --git a/tests/manifests/archive-members.manifest b/tests/manifests/archive-members.manifest
new file mode 100644
index 0000000..eee336a
--- /dev/null
+++ b/tests/manifests/archive-members.manifest
@@ -0,0 +1,12 @@
+BatchEngine.o
+Contact.o
+ContactBook.o
+Factory.o
+FormatPipeline.o
+Formatter.o
+RpnEvaluator.o
+RuntimeType.o
+ScalarConverter.o
+ScalarLiteral.o
+Serializer.o
+TextBuffer.o
diff --git a/tests/manifests/archive-symbols.manifest b/tests/manifests/archive-symbols.manifest
new file mode 100644
index 0000000..69063a0
--- /dev/null
+++ b/tests/manifests/archive-symbols.manifest
@@ -0,0 +1,104 @@
+cppf::BatchEngine::replace(std::basic_istream<char, std::char_traits<char>>&)
+cppf::BatchEngine::results() const
+cppf::BatchEngine::write(std::basic_ostream<char, std::char_traits<char>>&) const
+cppf::Contact::Contact()
+cppf::Contact::Contact(std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&, std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&)
+cppf::Contact::empty() const
+cppf::Contact::name() const
+cppf::Contact::note() const
+cppf::Contact::swap(cppf::Contact&)
+cppf::ContactBook::ContactBook()
+cppf::ContactBook::add(cppf::Contact const&)
+cppf::ContactBook::at(unsigned long) const
+cppf::ContactBook::size() const
+cppf::ContactBook::write(std::basic_ostream<char, std::char_traits<char>>&) const
+cppf::DefaultFormatterCreator::create(std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&) const
+cppf::FormatPipeline::FormatPipeline()
+cppf::FormatPipeline::FormatPipeline(cppf::FormatPipeline const&)
+cppf::FormatPipeline::append(cppf::Formatter const&)
+cppf::FormatPipeline::apply(cppf::TextBuffer const&) const
+cppf::FormatPipeline::operator=(cppf::FormatPipeline const&)
+cppf::FormatPipeline::size() const
+cppf::FormatPipeline::swap(cppf::FormatPipeline&)
+cppf::FormatPipeline::~FormatPipeline()
+cppf::Formatter::~Formatter()
+cppf::FormatterCreator::~FormatterCreator()
+cppf::InvalidScalar::what() const
+cppf::InvalidSpecification::what() const
+cppf::JobResult::JobResult()
+cppf::JobResult::JobResult(std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&, long)
+cppf::JobResult::name() const
+cppf::JobResult::value() const
+cppf::Payload::Payload(unsigned long, std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&)
+cppf::PipelineBuilder::replace(cppf::FormatPipeline&, cppf::FormatterCreator const&, std::basic_string<char, std::char_traits<char>, std::allocator<char>> const*, unsigned long)
+cppf::PrefixFormatter::PrefixFormatter(cppf::TextBuffer const&)
+cppf::PrefixFormatter::apply(cppf::TextBuffer const&) const
+cppf::PrefixFormatter::clone() const
+cppf::PrefixFormatter::name() const
+cppf::RpnEvaluator::evaluate(std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&)
+cppf::RuntimeBase::RuntimeBase()
+cppf::RuntimeBase::~RuntimeBase()
+cppf::RuntimeInspector::create(cppf::RuntimeKind)
+cppf::RuntimeInspector::identify(cppf::RuntimeBase const&)
+cppf::RuntimeInspector::identify(cppf::RuntimeBase const*)
+cppf::RuntimeInspector::name(cppf::RuntimeKind)
+cppf::ScalarConverter::write(std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&, std::basic_ostream<char, std::char_traits<char>>&)
+cppf::Serializer::deserialize(unsigned long)
+cppf::Serializer::serialize(cppf::Payload*)
+cppf::SuffixFormatter::SuffixFormatter(cppf::TextBuffer const&)
+cppf::SuffixFormatter::apply(cppf::TextBuffer const&) const
+cppf::SuffixFormatter::clone() const
+cppf::SuffixFormatter::name() const
+cppf::TextBuffer::TextBuffer()
+cppf::TextBuffer::TextBuffer(char const*)
+cppf::TextBuffer::TextBuffer(cppf::TextBuffer const&)
+cppf::TextBuffer::at(unsigned long)
+cppf::TextBuffer::at(unsigned long) const
+cppf::TextBuffer::c_str() const
+cppf::TextBuffer::empty() const
+cppf::TextBuffer::operator+=(cppf::TextBuffer const&)
+cppf::TextBuffer::operator=(cppf::TextBuffer const&)
+cppf::TextBuffer::size() const
+cppf::TextBuffer::swap(cppf::TextBuffer&)
+cppf::TextBuffer::~TextBuffer()
+cppf::UnknownFormatter::what() const
+cppf::UppercaseFormatter::apply(cppf::TextBuffer const&) const
+cppf::UppercaseFormatter::clone() const
+cppf::UppercaseFormatter::name() const
+cppf::operator!=(cppf::TextBuffer const&, cppf::TextBuffer const&)
+cppf::operator+(cppf::TextBuffer const&, cppf::TextBuffer const&)
+cppf::operator<(cppf::TextBuffer const&, cppf::TextBuffer const&)
+cppf::operator<<(std::basic_ostream<char, std::char_traits<char>>&, cppf::TextBuffer const&)
+cppf::operator==(cppf::JobResult const&, cppf::JobResult const&)
+cppf::operator==(cppf::TextBuffer const&, cppf::TextBuffer const&)
+cppf::scalar_detail::parseScalarLiteral(std::basic_string<char, std::char_traits<char>, std::allocator<char>> const&)
+typeinfo for cppf::DefaultFormatterCreator
+typeinfo for cppf::Formatter
+typeinfo for cppf::FormatterCreator
+typeinfo for cppf::InvalidScalar
+typeinfo for cppf::InvalidSpecification
+typeinfo for cppf::PrefixFormatter
+typeinfo for cppf::RuntimeBase
+typeinfo for cppf::SuffixFormatter
+typeinfo for cppf::UnknownFormatter
+typeinfo for cppf::UppercaseFormatter
+typeinfo name for cppf::DefaultFormatterCreator
+typeinfo name for cppf::Formatter
+typeinfo name for cppf::FormatterCreator
+typeinfo name for cppf::InvalidScalar
+typeinfo name for cppf::InvalidSpecification
+typeinfo name for cppf::PrefixFormatter
+typeinfo name for cppf::RuntimeBase
+typeinfo name for cppf::SuffixFormatter
+typeinfo name for cppf::UnknownFormatter
+typeinfo name for cppf::UppercaseFormatter
+vtable for cppf::DefaultFormatterCreator
+vtable for cppf::Formatter
+vtable for cppf::FormatterCreator
+vtable for cppf::InvalidScalar
+vtable for cppf::InvalidSpecification
+vtable for cppf::PrefixFormatter
+vtable for cppf::RuntimeBase
+vtable for cppf::SuffixFormatter
+vtable for cppf::UnknownFormatter
+vtable for cppf::UppercaseFormatter
diff --git a/tests/manifests/macho-libraries.manifest b/tests/manifests/macho-libraries.manifest
new file mode 100644
index 0000000..8e573c3
--- /dev/null
+++ b/tests/manifests/macho-libraries.manifest
@@ -0,0 +1,2 @@
+/usr/lib/libSystem.B.dylib
+/usr/lib/libc++.1.dylib


## `test(release): 실행 결정성과 메모리 해제 검증`

diff --git a/Makefile b/Makefile
index 392c210..7d51f80 100644
--- a/Makefile
+++ b/Makefile
@@ -38,7 +38,8 @@ PUBLIC_CONTRACT_SRC := tests/integration/test_public_contract.cpp
 RELEASE_BIN := $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
-	test-integration check-archive check-dependencies test check clean fclean re
+	test-integration test-leak check-archive check-dependencies \
+	check-determinism test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -161,9 +162,17 @@ test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 check-archive: $(NAME)
 	sh tests/check_archive.sh $(NAME)
 
+test-leak: $(TEST_BIN) $(NO_ELIDE_BIN) $(PUBLIC_CONTRACT_BIN)
+	sh tests/check_leaks.sh $(TEST_BIN) $(NO_ELIDE_BIN) \
+		$(PUBLIC_CONTRACT_BIN)
+
 check-dependencies: $(RELEASE_BIN)
 	sh tests/check_dependencies.sh $(RELEASE_BIN)
 
+check-determinism: $(APP_BIN)
+	LC_ALL=C LANG=C TZ=UTC sh tests/check_cli.sh
+	LC_ALL=C LANG=C TZ=UTC sh tests/check_cli.sh
+
 test: test-unit failure-test test-no-elide test-contract test-integration
 
 check:
@@ -173,6 +182,8 @@ check:
 	$(MAKE) test
 	$(MAKE) check-archive
 	$(MAKE) check-dependencies
+	$(MAKE) check-determinism
+	$(MAKE) test-leak
 	$(MAKE) -q all
 
 clean:
diff --git a/tests/check_leaks.sh b/tests/check_leaks.sh
new file mode 100644
index 0000000..64ff9bf
--- /dev/null
+++ b/tests/check_leaks.sh
@@ -0,0 +1,21 @@
+#!/bin/sh
+
+set -eu
+
+if [ "$#" -eq 0 ] || [ "$(/usr/bin/uname -s)" != "Darwin" ] ||
+    [ ! -x /usr/bin/leaks ]
+then
+    printf '누수 검사는 macOS leaks 도구와 실행 파일이 필요합니다.\n' >&2
+    exit 2
+fi
+
+for binary in "$@"
+do
+    if [ ! -x "$binary" ]
+    then
+        printf '누수 검사 실행 파일을 찾을 수 없습니다: %s\n' \
+            "$binary" >&2
+        exit 2
+    fi
+    /usr/bin/leaks -q --atExit -- "$binary"
+done


## `build(check): undefined behavior 검사 대상 추가`

diff --git a/Makefile b/Makefile
index 7d51f80..7f8d786 100644
--- a/Makefile
+++ b/Makefile
@@ -35,11 +35,14 @@ BATCH_FAILURE_SRC := tests/failure/test_batch_failure.cpp \
 NO_ELIDE_BIN := build/tests/unit_no_elide
 PUBLIC_CONTRACT_BIN := build/tests/public_contract
 PUBLIC_CONTRACT_SRC := tests/integration/test_public_contract.cpp
+SANITIZER_BIN := build/tests/unit_sanitize
+SANITIZER_FLAGS := -O1 -fsanitize=undefined -fno-sanitize-recover=all \
+	-fno-omit-frame-pointer -g
 RELEASE_BIN := $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
-	test-integration test-leak check-archive check-dependencies \
-	check-determinism test check clean fclean re
+	test-integration test-sanitize test-leak check-archive \
+	check-dependencies check-determinism test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
 
@@ -159,13 +162,22 @@ test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 	sh tests/check_cli.sh
 	./$(PUBLIC_CONTRACT_BIN)
 
-check-archive: $(NAME)
-	sh tests/check_archive.sh $(NAME)
+$(SANITIZER_BIN): $(SRC) $(TEST_SRC) $(TEST_SUPPORT_SRC)
+	@$(MKDIR) $(dir $@)
+	$(CXX) $(CPPFLAGS) $(CXXFLAGS) $(SANITIZER_FLAGS) \
+		$(SRC) $(TEST_SRC) $(TEST_SUPPORT_SRC) -o $@
+
+test-sanitize: $(SANITIZER_BIN)
+	UBSAN_OPTIONS=halt_on_error=1:print_stacktrace=1 \
+		./$(SANITIZER_BIN)
 
 test-leak: $(TEST_BIN) $(NO_ELIDE_BIN) $(PUBLIC_CONTRACT_BIN)
 	sh tests/check_leaks.sh $(TEST_BIN) $(NO_ELIDE_BIN) \
 		$(PUBLIC_CONTRACT_BIN)
 
+check-archive: $(NAME)
+	sh tests/check_archive.sh $(NAME)
+
 check-dependencies: $(RELEASE_BIN)
 	sh tests/check_dependencies.sh $(RELEASE_BIN)
 
@@ -183,6 +195,7 @@ check:
 	$(MAKE) check-archive
 	$(MAKE) check-dependencies
 	$(MAKE) check-determinism
+	$(MAKE) test-sanitize
 	$(MAKE) test-leak
 	$(MAKE) -q all
 


## `test(consumer): 저장소 밖 공개 library 연결 검증`

diff --git a/Makefile b/Makefile
index 10035cd..1bf8937 100644
--- a/Makefile
+++ b/Makefile
@@ -47,7 +47,7 @@ SANITIZER_FLAGS := -O1 -fsanitize=undefined -fno-sanitize-recover=all \
 RELEASE_BIN := $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
 
 .PHONY: all test-unit failure-test test-no-elide test-contract \
-	test-integration test-sanitize test-leak check-archive \
+	test-integration test-consumer test-sanitize test-leak check-archive \
 	check-dependencies check-determinism test check clean fclean re
 
 all: $(NAME) $(APP_BIN)
@@ -175,7 +175,10 @@ test-contract:
 	@! $(CXX) $(PUBLIC_CPPFLAGS) $(CXXFLAGS) -fsyntax-only \
 		tests/compile/template_list_sort_fail.cpp >/dev/null 2>&1
 
-test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN)
+test-consumer: $(NAME)
+	sh tests/check_external_consumer.sh "$(CXX)" "$(abspath $(NAME))"
+
+test-integration: $(APP_BIN) $(PUBLIC_CONTRACT_BIN) test-consumer
 	sh tests/check_cli.sh
 	./$(PUBLIC_CONTRACT_BIN)
 
diff --git a/tests/check_external_consumer.sh b/tests/check_external_consumer.sh
new file mode 100755
index 0000000..8b57cf9
--- /dev/null
+++ b/tests/check_external_consumer.sh
@@ -0,0 +1,44 @@
+#!/bin/sh
+
+set -eu
+
+if [ "$#" -ne 2 ] || [ ! -f "$2" ]
+then
+    printf '사용할 C++ 컴파일러와 정적 라이브러리가 필요합니다.\n' >&2
+    exit 2
+fi
+
+compiler=$1
+archive=$2
+project_root=$(CDPATH= cd "$(dirname "$0")/.." && pwd)
+temporary_directory=$(mktemp -d \
+    "${TMPDIR:-/tmp}/cpp-foundation-consumer.XXXXXX")
+
+cleanup()
+{
+    rm -f "$temporary_directory/main.cpp" \
+        "$temporary_directory/consumer"
+    rmdir "$temporary_directory"
+}
+
+trap cleanup EXIT HUP INT TERM
+
+if ! command -v "$compiler" >/dev/null 2>&1
+then
+    printf 'C++ 컴파일러를 찾을 수 없습니다: %s\n' "$compiler" >&2
+    exit 2
+fi
+
+cp "$project_root/tests/consumer/external_main.cpp" \
+    "$temporary_directory/main.cpp"
+"$compiler" -I"$project_root/include" \
+    -Wall -Wextra -Werror -Wpedantic -pedantic-errors -std=c++98 \
+    -Wold-style-cast -Wcast-qual -Woverloaded-virtual \
+    -Wnon-virtual-dtor \
+    "$temporary_directory/main.cpp" "$archive" \
+    -o "$temporary_directory/consumer"
+
+(
+    cd "$temporary_directory"
+    ./consumer
+)
diff --git a/tests/consumer/external_main.cpp b/tests/consumer/external_main.cpp
new file mode 100644
index 0000000..1aca950
--- /dev/null
+++ b/tests/consumer/external_main.cpp
@@ -0,0 +1,37 @@
+#include "cppf/Contact.hpp"
+#include "cppf/ContactBook.hpp"
+#include "cppf/FormatPipeline.hpp"
+#include "cppf/Formatter.hpp"
+#include "cppf/RpnEvaluator.hpp"
+#include "cppf/TextBuffer.hpp"
+
+#include <sstream>
+
+int main()
+{
+    cppf::ContactBook book;
+
+    book.add(cppf::Contact("Ada", "8 7 *"));
+    book.add(cppf::Contact("Grace", "9 7 *"));
+    if (book.size() != 2 || book.at(0).name() != "Ada")
+        return 1;
+
+    cppf::FormatPipeline pipeline;
+    const cppf::UppercaseFormatter uppercase;
+
+    pipeline.append(uppercase);
+    if (pipeline.apply(cppf::TextBuffer(book.at(0).name().c_str())) !=
+        cppf::TextBuffer("ADA"))
+        return 1;
+
+    if (cppf::RpnEvaluator::evaluate(book.at(0).note()) != 56 ||
+        cppf::RpnEvaluator::evaluate(book.at(1).note()) != 63)
+        return 1;
+
+    std::ostringstream output;
+
+    book.write(output);
+    if (output.str() != "0|Ada|8 7 *\n1|Grace|9 7 *\n")
+        return 1;
+    return 0;
+}


