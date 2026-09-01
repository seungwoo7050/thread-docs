# 다층 검증 게이트와 크로스플랫폼 CI

## `build(makefile): CXX98 검사 빌드 구성`

diff --git a/.gitignore b/.gitignore
new file mode 100644
index 0000000..7589170
--- /dev/null
+++ b/.gitignore
@@ -0,0 +1,6 @@
+build/
+*.o
+*.d
+*.out
+*.log
+.DS_Store
diff --git a/Makefile b/Makefile
new file mode 100644
index 0000000..ea63678
--- /dev/null
+++ b/Makefile
@@ -0,0 +1,28 @@
+CXX := c++
+CXXFLAGS := -Wall -Wextra -Werror -std=c++98
+CPPFLAGS := -Iinclude
+
+BUILD_DIR := build
+TEST_NAMES := test_containers
+TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
+HEADERS := $(wildcard include/*.hpp)
+
+.PHONY: all test clean fclean re
+
+all: $(TEST_BINS)
+
+$(BUILD_DIR):
+	mkdir -p $(BUILD_DIR)
+
+$(BUILD_DIR)/%: tests/%.cpp $(HEADERS) | $(BUILD_DIR)
+	$(CXX) $(CXXFLAGS) $(CPPFLAGS) $< -o $@
+
+test: $(TEST_BINS)
+	@for test_bin in $(TEST_BINS); do ./$$test_bin || exit $$?; done
+
+clean:
+	rm -rf $(BUILD_DIR)
+
+fclean: clean
+
+re: fclean all


## `test(headers): 공개 헤더를 각각 독립 compile`

diff --git a/Makefile b/Makefile
index a618968..e961121 100644
--- a/Makefile
+++ b/Makefile
@@ -10,20 +10,32 @@ TEST_NAMES := test_containers test_vector_exceptions test_map_exceptions \
 TEST_SUPPORT_HEADERS := $(wildcard tests/support/*.hpp)
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
 HEADERS := $(wildcard include/*.hpp)
+HEADER_TEST_SOURCES := $(wildcard tests/headers/*.cpp)
+HEADER_TEST_OBJECTS := $(patsubst tests/headers/%.cpp,\
+	$(BUILD_DIR)/headers/%.o,$(HEADER_TEST_SOURCES))
 
-.PHONY: all test clean fclean re
+.PHONY: all test headers clean fclean re
 
 all: $(TEST_BINS)
 
 $(BUILD_DIR):
 	mkdir -p $(BUILD_DIR)
 
+$(BUILD_DIR)/headers:
+	mkdir -p $@
+
 $(BUILD_DIR)/%: tests/%.cpp $(HEADERS) $(TEST_SUPPORT_HEADERS) | $(BUILD_DIR)
 	$(CXX) $(CXXFLAGS) $(CPPFLAGS) $< -o $@
 
+$(BUILD_DIR)/headers/%.o: tests/headers/%.cpp $(HEADERS) \
+		| $(BUILD_DIR)/headers
+	$(CXX) $(CXXFLAGS) $(CPPFLAGS) -c $< -o $@
+
 test: $(TEST_BINS)
 	@for test_bin in $(TEST_BINS); do ./$$test_bin || exit $$?; done
 
+headers: $(HEADER_TEST_OBJECTS)
+
 clean:
 	rm -rf $(BUILD_DIR)
 
diff --git a/tests/headers/ft_algorithm.cpp b/tests/headers/ft_algorithm.cpp
new file mode 100644
index 0000000..5fdf219
--- /dev/null
+++ b/tests/headers/ft_algorithm.cpp
@@ -0,0 +1,9 @@
+#include "ft_algorithm.hpp"
+
+int main()
+{
+	int lhs[] = {1, 2};
+	int rhs[] = {1, 3};
+	return ft::equal(lhs, lhs + 1, rhs)
+		&& ft::lexicographical_compare(lhs, lhs + 2, rhs, rhs + 2) ? 0 : 1;
+}
diff --git a/tests/headers/ft_containers.cpp b/tests/headers/ft_containers.cpp
new file mode 100644
index 0000000..baa83f4
--- /dev/null
+++ b/tests/headers/ft_containers.cpp
@@ -0,0 +1,11 @@
+#include "ft_containers.hpp"
+
+int main()
+{
+	ft::vector<int> values(2, 3);
+	ft::stack<int> pending;
+	ft::map<int, int> indexed;
+	pending.push(values.front());
+	indexed.insert(ft::make_pair(1, pending.top()));
+	return indexed.begin()->second == 3 ? 0 : 1;
+}
diff --git a/tests/headers/ft_iterator.cpp b/tests/headers/ft_iterator.cpp
new file mode 100644
index 0000000..fc02770
--- /dev/null
+++ b/tests/headers/ft_iterator.cpp
@@ -0,0 +1,8 @@
+#include "ft_iterator.hpp"
+
+int main()
+{
+	int values[] = {4, 5};
+	ft::reverse_iterator<int*> iterator(values + 2);
+	return *iterator == 5 ? 0 : 1;
+}
diff --git a/tests/headers/ft_map.cpp b/tests/headers/ft_map.cpp
new file mode 100644
index 0000000..45f1913
--- /dev/null
+++ b/tests/headers/ft_map.cpp
@@ -0,0 +1,8 @@
+#include "ft_map.hpp"
+
+int main()
+{
+	ft::map<int, int> values;
+	values.insert(ft::make_pair(2, 6));
+	return values.find(2)->second == 6 ? 0 : 1;
+}
diff --git a/tests/headers/ft_pair.cpp b/tests/headers/ft_pair.cpp
new file mode 100644
index 0000000..a433aa0
--- /dev/null
+++ b/tests/headers/ft_pair.cpp
@@ -0,0 +1,7 @@
+#include "ft_pair.hpp"
+
+int main()
+{
+	ft::pair<int, int> value = ft::make_pair(2, 7);
+	return value.first == 2 && value.second == 7 ? 0 : 1;
+}
diff --git a/tests/headers/ft_stack.cpp b/tests/headers/ft_stack.cpp
new file mode 100644
index 0000000..7900707
--- /dev/null
+++ b/tests/headers/ft_stack.cpp
@@ -0,0 +1,8 @@
+#include "ft_stack.hpp"
+
+int main()
+{
+	ft::stack<int> values;
+	values.push(9);
+	return values.top() == 9 ? 0 : 1;
+}
diff --git a/tests/headers/ft_type_traits.cpp b/tests/headers/ft_type_traits.cpp
new file mode 100644
index 0000000..a561cdf
--- /dev/null
+++ b/tests/headers/ft_type_traits.cpp
@@ -0,0 +1,7 @@
+#include "ft_type_traits.hpp"
+
+int main()
+{
+	return ft::is_integral<int>::value
+		&& !ft::is_integral<float>::value ? 0 : 1;
+}
diff --git a/tests/headers/ft_vector.cpp b/tests/headers/ft_vector.cpp
new file mode 100644
index 0000000..8775bae
--- /dev/null
+++ b/tests/headers/ft_vector.cpp
@@ -0,0 +1,7 @@
+#include "ft_vector.hpp"
+
+int main()
+{
+	ft::vector<int> values(3, 5);
+	return values.size() == 3 && values.back() == 5 ? 0 : 1;
+}


## `test(consumer): 다중 번역 단위 공개 헤더 사용 검증`

diff --git a/Makefile b/Makefile
index e961121..d422fd4 100644
--- a/Makefile
+++ b/Makefile
@@ -13,8 +13,12 @@ HEADERS := $(wildcard include/*.hpp)
 HEADER_TEST_SOURCES := $(wildcard tests/headers/*.cpp)
 HEADER_TEST_OBJECTS := $(patsubst tests/headers/%.cpp,\
 	$(BUILD_DIR)/headers/%.o,$(HEADER_TEST_SOURCES))
+CONSUMER_SOURCES := $(wildcard tests/consumer/*.cpp)
+CONSUMER_OBJECTS := $(patsubst tests/consumer/%.cpp,\
+	$(BUILD_DIR)/consumer/%.o,$(CONSUMER_SOURCES))
+CONSUMER_BIN := $(BUILD_DIR)/consumer_test
 
-.PHONY: all test headers clean fclean re
+.PHONY: all test headers consumer check clean fclean re
 
 all: $(TEST_BINS)
 
@@ -24,6 +28,9 @@ $(BUILD_DIR):
 $(BUILD_DIR)/headers:
 	mkdir -p $@
 
+$(BUILD_DIR)/consumer:
+	mkdir -p $@
+
 $(BUILD_DIR)/%: tests/%.cpp $(HEADERS) $(TEST_SUPPORT_HEADERS) | $(BUILD_DIR)
 	$(CXX) $(CXXFLAGS) $(CPPFLAGS) $< -o $@
 
@@ -31,11 +38,23 @@ $(BUILD_DIR)/headers/%.o: tests/headers/%.cpp $(HEADERS) \
 		| $(BUILD_DIR)/headers
 	$(CXX) $(CXXFLAGS) $(CPPFLAGS) -c $< -o $@
 
+$(BUILD_DIR)/consumer/%.o: tests/consumer/%.cpp $(HEADERS) \
+		tests/consumer/consumer_api.hpp | $(BUILD_DIR)/consumer
+	$(CXX) $(CXXFLAGS) $(CPPFLAGS) -c $< -o $@
+
+$(CONSUMER_BIN): $(CONSUMER_OBJECTS)
+	$(CXX) $(CXXFLAGS) $(CONSUMER_OBJECTS) -o $@
+
 test: $(TEST_BINS)
 	@for test_bin in $(TEST_BINS); do ./$$test_bin || exit $$?; done
 
 headers: $(HEADER_TEST_OBJECTS)
 
+consumer: $(CONSUMER_BIN)
+	./$(CONSUMER_BIN)
+
+check: test headers consumer
+
 clean:
 	rm -rf $(BUILD_DIR)
 
diff --git a/tests/consumer/consumer_api.hpp b/tests/consumer/consumer_api.hpp
new file mode 100644
index 0000000..82dceb3
--- /dev/null
+++ b/tests/consumer/consumer_api.hpp
@@ -0,0 +1,7 @@
+#ifndef TESTS_CONSUMER_CONSUMER_API_HPP
+# define TESTS_CONSUMER_CONSUMER_API_HPP
+
+int vector_consumer_result();
+int map_consumer_result();
+
+#endif
diff --git a/tests/consumer/main.cpp b/tests/consumer/main.cpp
new file mode 100644
index 0000000..03211ce
--- /dev/null
+++ b/tests/consumer/main.cpp
@@ -0,0 +1,20 @@
+#include "consumer_api.hpp"
+
+#include <cstdlib>
+#include <iostream>
+
+int main()
+{
+	if (vector_consumer_result() != 29)
+	{
+		std::cerr << "FAIL: vector consumer result" << std::endl;
+		return EXIT_FAILURE;
+	}
+	if (map_consumer_result() != 55)
+	{
+		std::cerr << "FAIL: map consumer result" << std::endl;
+		return EXIT_FAILURE;
+	}
+	std::cout << "multi-translation-unit consumer checks passed" << std::endl;
+	return EXIT_SUCCESS;
+}
diff --git a/tests/consumer/map_consumer.cpp b/tests/consumer/map_consumer.cpp
new file mode 100644
index 0000000..aa8f3d4
--- /dev/null
+++ b/tests/consumer/map_consumer.cpp
@@ -0,0 +1,16 @@
+#include "ft_map.hpp"
+#include "consumer_api.hpp"
+
+int map_consumer_result()
+{
+	ft::map<int, int> values;
+	values.insert(ft::make_pair(3, 30));
+	values.insert(ft::make_pair(1, 10));
+	values.insert(ft::make_pair(2, 20));
+	values.erase(1);
+	int result = 0;
+	for (ft::map<int, int>::const_iterator it = values.begin();
+		it != values.end(); ++it)
+		result += it->first + it->second;
+	return result;
+}
diff --git a/tests/consumer/vector_consumer.cpp b/tests/consumer/vector_consumer.cpp
new file mode 100644
index 0000000..8a29000
--- /dev/null
+++ b/tests/consumer/vector_consumer.cpp
@@ -0,0 +1,15 @@
+#include "ft_vector.hpp"
+#include "consumer_api.hpp"
+
+int vector_consumer_result()
+{
+	ft::vector<int> values;
+	for (int value = 1; value <= 5; ++value)
+		values.push_back(value);
+	values.insert(values.begin() + 2, 2, 7);
+	int result = 0;
+	for (ft::vector<int>::const_iterator it = values.begin();
+		it != values.end(); ++it)
+		result += *it;
+	return result;
+}


## `build(makefile): 격리된 sanitizer 검사 대상 추가`

diff --git a/Makefile b/Makefile
index d422fd4..20ac862 100644
--- a/Makefile
+++ b/Makefile
@@ -1,6 +1,8 @@
 CXX := c++
 CXXFLAGS := -Wall -Wextra -Werror -std=c++98
 CPPFLAGS := -Iinclude
+SANITIZER_FLAGS := -O1 -g -fno-omit-frame-pointer \
+	-fsanitize=address,undefined
 
 BUILD_DIR := build
 TEST_NAMES := test_containers test_vector_exceptions test_map_exceptions \
@@ -18,7 +20,7 @@ CONSUMER_OBJECTS := $(patsubst tests/consumer/%.cpp,\
 	$(BUILD_DIR)/consumer/%.o,$(CONSUMER_SOURCES))
 CONSUMER_BIN := $(BUILD_DIR)/consumer_test
 
-.PHONY: all test headers consumer check clean fclean re
+.PHONY: all test headers consumer check sanitize clean fclean re
 
 all: $(TEST_BINS)
 
@@ -55,6 +57,10 @@ consumer: $(CONSUMER_BIN)
 
 check: test headers consumer
 
+sanitize:
+	$(MAKE) BUILD_DIR=$(BUILD_DIR)/sanitize \
+		CXXFLAGS="$(CXXFLAGS) $(SANITIZER_FLAGS)" check
+
 clean:
 	rm -rf $(BUILD_DIR)
 


## `ci: compiler 행렬과 sanitizer 검사 구성`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
new file mode 100644
index 0000000..3876ec1
--- /dev/null
+++ b/.github/workflows/ci.yml
@@ -0,0 +1,38 @@
+name: CXX98 checks
+
+on:
+  push:
+  pull_request:
+
+permissions:
+  contents: read
+
+jobs:
+  compiler-matrix:
+    name: ${{ matrix.os }} / ${{ matrix.compiler }}
+    runs-on: ${{ matrix.os }}
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - os: ubuntu-latest
+            compiler: g++
+          - os: ubuntu-latest
+            compiler: clang++
+          - os: macos-latest
+            compiler: clang++
+    steps:
+      - uses: actions/checkout@v4
+      - name: Build and run checks
+        run: make CXX=${{ matrix.compiler }} check
+
+  sanitizers:
+    name: Linux sanitizers
+    runs-on: ubuntu-latest
+    env:
+      ASAN_OPTIONS: detect_leaks=1:halt_on_error=1
+      UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
+    steps:
+      - uses: actions/checkout@v4
+      - name: Run AddressSanitizer and UndefinedBehaviorSanitizer
+        run: make CXX=clang++ sanitize


## `ci: harden cross-platform verification`

diff --git a/.github/workflows/ci.yml b/.github/workflows/ci.yml
deleted file mode 100644
index 3876ec1..0000000
--- a/.github/workflows/ci.yml
+++ /dev/null
@@ -1,38 +0,0 @@
-name: CXX98 checks
-
-on:
-  push:
-  pull_request:
-
-permissions:
-  contents: read
-
-jobs:
-  compiler-matrix:
-    name: ${{ matrix.os }} / ${{ matrix.compiler }}
-    runs-on: ${{ matrix.os }}
-    strategy:
-      fail-fast: false
-      matrix:
-        include:
-          - os: ubuntu-latest
-            compiler: g++
-          - os: ubuntu-latest
-            compiler: clang++
-          - os: macos-latest
-            compiler: clang++
-    steps:
-      - uses: actions/checkout@v4
-      - name: Build and run checks
-        run: make CXX=${{ matrix.compiler }} check
-
-  sanitizers:
-    name: Linux sanitizers
-    runs-on: ubuntu-latest
-    env:
-      ASAN_OPTIONS: detect_leaks=1:halt_on_error=1
-      UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
-    steps:
-      - uses: actions/checkout@v4
-      - name: Run AddressSanitizer and UndefinedBehaviorSanitizer
-        run: make CXX=clang++ sanitize
diff --git a/.github/workflows/cpp-ft-container-ci.yml b/.github/workflows/cpp-ft-container-ci.yml
new file mode 100644
index 0000000..8221259
--- /dev/null
+++ b/.github/workflows/cpp-ft-container-ci.yml
@@ -0,0 +1,84 @@
+name: C++98 container CI
+
+on:
+  push:
+    branches:
+      - cpp/ft_container
+  pull_request:
+    branches:
+      - cpp/ft_container
+
+permissions:
+  contents: read
+
+concurrency:
+  group: ${{ github.workflow }}-${{ github.event.pull_request.number || github.ref }}
+  cancel-in-progress: true
+
+env:
+  LANG: C
+  LC_ALL: C
+  TZ: UTC
+
+jobs:
+  portability:
+    name: ${{ matrix.label }}
+    runs-on: ${{ matrix.os }}
+    timeout-minutes: 15
+    strategy:
+      fail-fast: false
+      matrix:
+        include:
+          - label: Ubuntu 24.04 / GCC 14
+            os: ubuntu-24.04
+            cxx: g++-14
+          - label: Ubuntu 24.04 / Clang 18
+            os: ubuntu-24.04
+            cxx: clang++-18
+          - label: macOS 15 / Apple Clang
+            os: macos-15
+            cxx: clang++
+    steps:
+      - name: Check out the project branch
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: |
+          set -euo pipefail
+          uname -a
+          command -v "$CXX_COMMAND"
+          "$CXX_COMMAND" --version
+          make --version
+
+      - name: Run container regression gates
+        env:
+          CXX_COMMAND: ${{ matrix.cxx }}
+        run: make check CXX="$CXX_COMMAND"
+
+  sanitizers:
+    name: Ubuntu 24.04 / Clang 18 / ASan and UBSan
+    runs-on: ubuntu-24.04
+    timeout-minutes: 15
+    env:
+      ASAN_OPTIONS: detect_leaks=1:halt_on_error=1:abort_on_error=1
+      UBSAN_OPTIONS: halt_on_error=1:print_stacktrace=1
+    steps:
+      - name: Check out the project branch
+        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
+        with:
+          persist-credentials: false
+
+      - name: Report toolchain
+        run: |
+          set -euo pipefail
+          uname -a
+          command -v clang++-18
+          clang++-18 --version
+          make --version
+
+      - name: Run combined sanitizer gate
+        run: make sanitize CXX=clang++-18
