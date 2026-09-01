# 컨테이너 조합과 헤더 전용 공개 계약

## `docs(readme): C++98 컨테이너 개발 기준 정의`

diff --git a/README.md b/README.md
new file mode 100644
index 0000000..59397c3
--- /dev/null
+++ b/README.md
@@ -0,0 +1,11 @@
+# ft_container
+
+C++98 환경에서 표준 컨테이너의 핵심 동작과 공개 인터페이스를 단계적으로 구현한다.
+
+## 개발 기준
+
+- 기능은 독립적으로 검토할 수 있는 작은 책임 단위로 추가한다.
+- 공개 헤더는 저장소 밖의 소비자도 사용할 수 있도록 자체 완결성을 유지한다.
+- iterator, allocator, 비교 연산의 계약을 구현과 함께 검증한다.
+- 빌드 산출물과 개인 환경 파일은 버전 관리에 포함하지 않는다.
+- 각 단계는 C++98 엄격 경고 설정에서 컴파일 가능한 상태로 남긴다.


## `feat(stack): vector 기반 stack 어댑터 구현`

diff --git a/include/ft_stack.hpp b/include/ft_stack.hpp
new file mode 100644
index 0000000..b43b2fa
--- /dev/null
+++ b/include/ft_stack.hpp
@@ -0,0 +1,82 @@
+#ifndef FT_STACK_HPP
+# define FT_STACK_HPP
+
+# include "ft_vector.hpp"
+
+namespace ft
+{
+	template <class T, class Container = ft::vector<T> >
+	class stack
+	{
+	public:
+		typedef Container container_type;
+		typedef typename container_type::value_type value_type;
+		typedef typename container_type::size_type size_type;
+
+		explicit stack(const container_type& cont = container_type())
+			: c(cont)
+		{
+		}
+
+		bool empty() const { return c.empty(); }
+		size_type size() const { return c.size(); }
+		value_type& top() { return c.back(); }
+		const value_type& top() const { return c.back(); }
+		void push(const value_type& value) { c.push_back(value); }
+		void pop() { c.pop_back(); }
+
+	protected:
+		container_type c;
+
+		template <class U, class C>
+		friend bool operator==(const stack<U, C>& lhs,
+			const stack<U, C>& rhs);
+		template <class U, class C>
+		friend bool operator<(const stack<U, C>& lhs,
+			const stack<U, C>& rhs);
+	};
+
+	template <class T, class Container>
+	bool operator==(const stack<T, Container>& lhs,
+		const stack<T, Container>& rhs)
+	{
+		return lhs.c == rhs.c;
+	}
+
+	template <class T, class Container>
+	bool operator!=(const stack<T, Container>& lhs,
+		const stack<T, Container>& rhs)
+	{
+		return !(lhs == rhs);
+	}
+
+	template <class T, class Container>
+	bool operator<(const stack<T, Container>& lhs,
+		const stack<T, Container>& rhs)
+	{
+		return lhs.c < rhs.c;
+	}
+
+	template <class T, class Container>
+	bool operator<=(const stack<T, Container>& lhs,
+		const stack<T, Container>& rhs)
+	{
+		return !(rhs < lhs);
+	}
+
+	template <class T, class Container>
+	bool operator>(const stack<T, Container>& lhs,
+		const stack<T, Container>& rhs)
+	{
+		return rhs < lhs;
+	}
+
+	template <class T, class Container>
+	bool operator>=(const stack<T, Container>& lhs,
+		const stack<T, Container>& rhs)
+	{
+		return !(lhs < rhs);
+	}
+}
+
+#endif


## `test(stack): 기본 동작과 관계 연산 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 89e2482..aaee589 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -1,5 +1,6 @@
 #include <cstdlib>
 #include <iostream>
+#include <stack>
 #include <stdexcept>
 #include <string>
 #include <vector>
@@ -7,6 +8,7 @@
 #include "ft_algorithm.hpp"
 #include "ft_iterator.hpp"
 #include "ft_pair.hpp"
+#include "ft_stack.hpp"
 #include "ft_type_traits.hpp"
 #include "ft_vector.hpp"
 
@@ -86,12 +88,49 @@ namespace
 		catch (const std::out_of_range&) { std_thrown = true; }
 		require(ft_thrown == std_thrown, "vector at out_of_range");
 	}
+
+	void test_stack()
+	{
+		ft::stack<int> fts;
+		std::stack<int, std::vector<int> > stds;
+		for (int i = 0; i < 12; ++i)
+		{
+			fts.push(i);
+			stds.push(i);
+			require(fts.top() == stds.top(), "stack top after push");
+		}
+		while (!stds.empty())
+		{
+			require(fts.size() == stds.size(), "stack size");
+			require(fts.top() == stds.top(), "stack top");
+			fts.pop();
+			stds.pop();
+		}
+		require(fts.empty() == stds.empty(), "stack empty");
+
+		ft::stack<int> fta;
+		ft::stack<int> ftb;
+		std::stack<int, std::vector<int> > stda;
+		std::stack<int, std::vector<int> > stdb;
+		for (int i = 0; i < 4; ++i)
+		{
+			fta.push(i);
+			ftb.push(i);
+			stda.push(i);
+			stdb.push(i);
+		}
+		ftb.push(9);
+		stdb.push(9);
+		require((fta == ftb) == (stda == stdb), "stack equality compare");
+		require((fta < ftb) == (stda < stdb), "stack less compare");
+	}
 }
 
 int main()
 {
 	test_utilities();
 	test_vector();
+	test_stack();
 	std::cout << "ft_containers checks passed" << std::endl;
 	return 0;
 }


## `feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가`

diff --git a/include/ft_containers.hpp b/include/ft_containers.hpp
new file mode 100644
index 0000000..fbcba73
--- /dev/null
+++ b/include/ft_containers.hpp
@@ -0,0 +1,11 @@
+#ifndef FT_CONTAINERS_HPP
+# define FT_CONTAINERS_HPP
+
+# include "ft_algorithm.hpp"
+# include "ft_iterator.hpp"
+# include "ft_pair.hpp"
+# include "ft_stack.hpp"
+# include "ft_type_traits.hpp"
+# include "ft_vector.hpp"
+
+#endif


## `test(headers): 통합 헤더의 순차 컨테이너 표면 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index aaee589..318e81e 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -5,12 +5,7 @@
 #include <string>
 #include <vector>
 
-#include "ft_algorithm.hpp"
-#include "ft_iterator.hpp"
-#include "ft_pair.hpp"
-#include "ft_stack.hpp"
-#include "ft_type_traits.hpp"
-#include "ft_vector.hpp"
+#include "ft_containers.hpp"
 
 namespace
 {


## `feat(map): 관계 연산과 통합 공개 헤더 완성`

diff --git a/include/ft_containers.hpp b/include/ft_containers.hpp
index fbcba73..e316d26 100644
--- a/include/ft_containers.hpp
+++ b/include/ft_containers.hpp
@@ -3,6 +3,7 @@
 
 # include "ft_algorithm.hpp"
 # include "ft_iterator.hpp"
+# include "ft_map.hpp"
 # include "ft_pair.hpp"
 # include "ft_stack.hpp"
 # include "ft_type_traits.hpp"
diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 86aa7d1..de89781 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -42,9 +42,27 @@ namespace ft
 			}
 		};
 
-	typedef typename allocator_type::template rebind<node>::other node_allocator;
+		typedef typename allocator_type::template rebind<node>::other node_allocator;
 
 	public:
+		class value_compare
+		{
+			friend class map;
+
+		public:
+			bool operator()(const value_type& lhs, const value_type& rhs) const
+			{
+				return comp(lhs.first, rhs.first);
+			}
+
+		protected:
+			key_compare comp;
+
+			explicit value_compare(key_compare c) : comp(c)
+			{
+			}
+		};
+
 		class iterator
 			: public ft::iterator<std::bidirectional_iterator_tag, value_type>
 		{
@@ -333,6 +351,7 @@ namespace ft
 		}
 
 		key_compare key_comp() const { return _comp; }
+		value_compare value_comp() const { return value_compare(_comp); }
 		allocator_type get_allocator() const { return _alloc; }
 
 		iterator find(const key_type& key)
@@ -550,6 +569,57 @@ namespace ft
 			_destroy_node(n);
 		}
 	};
+
+	template <class Key, class T, class Compare, class Alloc>
+	bool operator==(const map<Key, T, Compare, Alloc>& lhs,
+		const map<Key, T, Compare, Alloc>& rhs)
+	{
+		return lhs.size() == rhs.size()
+			&& ft::equal(lhs.begin(), lhs.end(), rhs.begin());
+	}
+
+	template <class Key, class T, class Compare, class Alloc>
+	bool operator!=(const map<Key, T, Compare, Alloc>& lhs,
+		const map<Key, T, Compare, Alloc>& rhs)
+	{
+		return !(lhs == rhs);
+	}
+
+	template <class Key, class T, class Compare, class Alloc>
+	bool operator<(const map<Key, T, Compare, Alloc>& lhs,
+		const map<Key, T, Compare, Alloc>& rhs)
+	{
+		return ft::lexicographical_compare(lhs.begin(), lhs.end(),
+			rhs.begin(), rhs.end());
+	}
+
+	template <class Key, class T, class Compare, class Alloc>
+	bool operator<=(const map<Key, T, Compare, Alloc>& lhs,
+		const map<Key, T, Compare, Alloc>& rhs)
+	{
+		return !(rhs < lhs);
+	}
+
+	template <class Key, class T, class Compare, class Alloc>
+	bool operator>(const map<Key, T, Compare, Alloc>& lhs,
+		const map<Key, T, Compare, Alloc>& rhs)
+	{
+		return rhs < lhs;
+	}
+
+	template <class Key, class T, class Compare, class Alloc>
+	bool operator>=(const map<Key, T, Compare, Alloc>& lhs,
+		const map<Key, T, Compare, Alloc>& rhs)
+	{
+		return !(lhs < rhs);
+	}
+
+	template <class Key, class T, class Compare, class Alloc>
+	void swap(map<Key, T, Compare, Alloc>& lhs,
+		map<Key, T, Compare, Alloc>& rhs)
+	{
+		lhs.swap(rhs);
+	}
 }
 
 #endif


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


## `docs: improve README with project visuals`

diff --git a/README.md b/README.md
index 59397c3..98a709c 100644
--- a/README.md
+++ b/README.md
@@ -1,11 +1,175 @@
-# ft_container
+# C++98 Containers
 
-C++98 환경에서 표준 컨테이너의 핵심 동작과 공개 인터페이스를 단계적으로 구현한다.
+C++98 템플릿으로 `vector`, `stack`, `map`과 기반 도구를 구현한 헤더 전용 컨테이너 라이브러리입니다.
 
-## 개발 기준
+공개 결과를 표준 컨테이너와 비교하는 데 그치지 않습니다. 객체 수명, allocator 사용, 예외 뒤 상태, 반복자 동작, 레드-블랙 트리 불변식과 소비자 컴파일 범위를 별도로 검사합니다.
 
-- 기능은 독립적으로 검토할 수 있는 작은 책임 단위로 추가한다.
-- 공개 헤더는 저장소 밖의 소비자도 사용할 수 있도록 자체 완결성을 유지한다.
-- iterator, allocator, 비교 연산의 계약을 구현과 함께 검증한다.
-- 빌드 산출물과 개인 환경 파일은 버전 관리에 포함하지 않는다.
-- 각 단계는 C++98 엄격 경고 설정에서 컴파일 가능한 상태로 남긴다.
+![vector 객체 수명과 map 레드-블랙 트리 불변식](docs/images/container-invariants.svg)
+
+## 한눈에 보기
+
+| 항목 | 내용 |
+| --- | --- |
+| 언어 | C++98 |
+| 배포 형식 | `include/`의 헤더 전용 라이브러리 |
+| 주요 컨테이너 | `ft::vector`, `ft::stack`, `ft::map` |
+| 기반 도구 | `pair`, iterator traits, reverse iterator, 비교 알고리즘, type traits |
+| `map` 표현 | 헤더 sentinel을 둔 레드-블랙 트리 |
+| 주요 검증 | 표준 결과 비교, 예외·수명, 반복자, 무작위 불변식, 복잡도, 소비자 빌드 |
+
+## 빠른 시작
+
+```sh
+make clean
+make check
+```
+
+모든 구성 요소를 사용하려면 통합 헤더를 포함합니다.
+
+```cpp
+#include "ft_containers.hpp"
+
+int main()
+{
+    ft::vector<int> numbers;
+    numbers.push_back(42);
+
+    ft::map<int, int> values;
+    values[1] = numbers.back();
+
+    ft::stack<int> pending;
+    pending.push(values[1]);
+    return pending.top() == 42 ? 0 : 1;
+}
+```
+
+```sh
+c++ -Wall -Wextra -Werror -std=c++98 -Iinclude example.cpp -o example
+./example
+```
+
+## 제공 범위
+
+| 헤더 | 공개 구성 요소 | 역할 |
+| --- | --- | --- |
+| `ft_containers.hpp` | 전체 공개 표면 | 컨테이너와 공용 도구의 통합 진입점 |
+| `ft_vector.hpp` | `ft::vector` | allocator 기반 연속 저장 컨테이너 |
+| `ft_stack.hpp` | `ft::stack` | 기본적으로 `ft::vector`를 사용하는 LIFO 어댑터 |
+| `ft_map.hpp` | `ft::map` | 레드-블랙 트리 기반 정렬 연관 컨테이너 |
+| `ft_pair.hpp` | `ft::pair`, `ft::make_pair` | 값 쌍과 관계 연산 |
+| `ft_iterator.hpp` | iterator traits, reverse iterator | 반복자 형식 정보와 역방향 순회 |
+| `ft_algorithm.hpp` | `equal`, `lexicographical_compare` | 범위 비교 |
+| `ft_type_traits.hpp` | `enable_if`, `integral_constant`, `is_integral` | 개수·범위 생성자와 과부하 선택 |
+
+표준 라이브러리 전체를 대체하지 않으며 README와 상세 문서에 명시한 C++98 인터페이스 부분집합만 제공합니다.
+
+## `vector` 표현과 수명
+
+`vector`는 allocator, 메모리 블록 포인터, 현재 크기와 용량을 직접 관리합니다.
+
+```text
+[data, data + size)       생성이 완료된 객체
+[data + size, data + cap) 아직 객체가 없는 저장 공간
+```
+
+다음 조건을 유지합니다.
+
+- 생성된 객체 수는 항상 `size`와 같습니다.
+- 메모리 블록 해제 전 생성된 모든 객체의 소멸자를 호출합니다.
+- 재할당은 새 블록의 원소 구성이 완료된 뒤 기존 상태를 교체합니다.
+- 범위 `assign`과 `insert`는 입력 snapshot을 만들어 자기 범위 별칭을 분리합니다.
+- 용량 증가는 대체로 두 배를 사용하되 `max_size()` 부근에서 상한을 넘지 않도록 계산합니다.
+
+재할당 없는 제자리 삽입·삭제에서 사용자 타입의 대입 연산자가 예외를 던지면 일부 원소 값은 이미 바뀔 수 있습니다. 이 경로 전체에 강한 예외 보장을 주장하지 않습니다.
+
+## `map` 표현과 불변식
+
+`map`은 값이 없는 헤더 sentinel과 allocator로 만든 값 노드를 분리합니다.
+
+```text
+header.parent -> root
+header.left   -> minimum
+header.right  -> maximum
+end()         -> header
+```
+
+삽입과 삭제는 회전과 색 보정을 수행하며 다음 조건을 확인합니다.
+
+- 루트는 검정입니다.
+- 빨강 노드의 자식은 검정입니다.
+- 모든 root-to-leaf 경로의 검정 높이가 같습니다.
+- 부모·자식 링크가 서로 일치합니다.
+- header가 root, minimum과 maximum을 올바르게 가리킵니다.
+- 도달 가능한 값 노드 수가 `size`와 같습니다.
+
+상태를 가진 비교자를 교환할 때는 비교자 교환을 트리 소유권 이동보다 먼저 수행합니다. 비교자가 상태를 바꾸기 전에 예외를 던지는 경로에서는 두 트리의 노드 소유권을 유지합니다.
+
+## 복잡도
+
+| 연산 | 시간 복잡도 | 비고 |
+| --- | ---: | --- |
+| `vector` 원소 접근·반복자·크기 조회 | O(1) | 각 연산의 사전 조건 적용 |
+| `vector::push_back` | 상각 O(1), 성장 시 O(n) | 재할당 시 새 블록 사용 |
+| `vector::insert`, `erase` | 이동 원소 수에 선형 | 범위 입력 snapshot은 O(m) 추가 공간 |
+| 기본 `stack` 조회·`top`·`pop` | O(1) | 다른 하위 컨테이너는 해당 비용 적용 |
+| `map` 검색·경계 질의·단일 삽입 | O(log n) | 비교자 비용 별도 |
+| `map` 범위 삽입·복사 | O(k log(n + k)) | 원소별 단일 삽입 |
+| `map::clear`, 관계 연산 | O(n) | 노드 정리 또는 원소열 비교 |
+| `map::swap` | O(log n + log m) | 교환 뒤 header 양 끝 갱신 |
+
+## 검증
+
+```sh
+make test
+make headers
+make consumer
+make check
+make CXX=clang++ sanitize
+```
+
+| 검사 | 확인하는 내용 |
+| --- | --- |
+| `test_containers.cpp` | 선택한 공개 결과를 `std` 컨테이너와 비교 |
+| `test_vector_exceptions.cpp` | 객체 수명, 할당 블록과 실패 뒤 상태 |
+| `test_map_exceptions.cpp` | 비교·노드 할당 실패와 트리 보존 |
+| `test_map_iterators.cpp` | 회전·삭제·교환 뒤 반복자 동작 |
+| `test_map_policy_exceptions.cpp` | 상태를 가진 비교자 대입 실패 |
+| `test_map_randomized.cpp` | 고정 seed 공개 결과와 레드-블랙 불변식 |
+| `test_complexity.cpp` | 트리 높이와 비교자 호출 상한 |
+| `tests/headers/` | 각 공개 헤더를 첫 include로 개별 컴파일 |
+| `tests/consumer/` | 여러 번역 단위에서 포함·링크·실행 |
+
+`make check`는 공개 동작, 예외, 반복자, 무작위 불변식, 복잡도, 헤더 독립성과 소비자 빌드를 모두 실행합니다. `make sanitize`는 별도 빌드 경로에서 AddressSanitizer와 UndefinedBehaviorSanitizer를 적용합니다.
+
+## 저장소 구성
+
+```text
+.
+├── include/                    # 공개 C++98 템플릿 헤더
+├── tests/                      # 공개·예외·구조·소비자 검사
+├── architecture/               # 구현 표현과 공개 계약
+├── .github/workflows/ci.yml    # compiler·sanitizer 행렬
+└── Makefile                    # 빌드와 검증 진입점
+```
+
+## 문서 읽기 순서
+
+1. [구현 의존 지도](architecture/implementation-dependency-map.md)
+2. [vector와 map 표현 불변식](architecture/vector-map-representation-invariants.md)
+3. [호환성·예외·반복자 계약](architecture/compatibility-exception-and-iterator-contract.md)
+4. [헤더 전용 소비자와 릴리스 범위](architecture/header-only-consumer-and-release-surface.md)
+
+## 제한 사항
+
+- 선택한 C++98 인터페이스만 제공하며 모든 표준 과부하를 포함하지 않습니다.
+- `map`의 hint 삽입은 일반 삽입으로 전달합니다.
+- `vector`의 일부 범위 변경은 입력 snapshot에 추가 공간을 사용합니다.
+- 제자리 변경 중 사용자 대입이 실패하면 일부 값이 이미 바뀔 수 있습니다.
+- `map::operator[]`은 `mapped_type`의 기본 생성을 요구합니다.
+- `is_integral`은 cv 한정 형식을 자동으로 정규화하지 않습니다.
+- allocator fancy pointer와 현대 C++ allocator 전파 정책 전체를 지원하지 않습니다.
+- 설치 패키지, ABI 버전 관리와 배포 메타데이터를 제공하지 않습니다.
+
+## 프로젝트 배경
+
+이 저장소는 42의 `ft_containers`에서 출발했습니다. `vector`, `stack`, `map` 구현을 유지하면서 객체 수명과 예외 경로, 반복자 안정성, 레드-블랙 트리 불변식, 복잡도 상한과 여러 번역 단위 소비자 검사를 독립적으로 추가했습니다.
diff --git a/docs/images/container-invariants.svg b/docs/images/container-invariants.svg
new file mode 100644
index 0000000..3d4e7ca
--- /dev/null
+++ b/docs/images/container-invariants.svg
@@ -0,0 +1,19 @@
+<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1200 600" role="img" aria-labelledby="title desc">
+  <title id="title">Vector lifetime and map tree invariants</title>
+  <desc id="desc">The left side separates constructed vector elements from unused capacity. The right side shows a map header sentinel connected to a red-black tree root, minimum, and maximum.</desc>
+  <defs><marker id="arrow" markerWidth="10" markerHeight="10" refX="8" refY="3" orient="auto"><path d="M0,0 L0,6 L9,3 z" fill="#68d6ff"/></marker>
+  <style>.bg{fill:#0b1020}.panel{fill:#151d31;stroke:#344463;stroke-width:2}.live{fill:#173b31;stroke:#6ee7a8;stroke-width:2}.space{fill:#222b40;stroke:#657594;stroke-width:2;stroke-dasharray:7 6}.node{fill:#17223a;stroke:#87a7ff;stroke-width:2}.red{fill:#3b1c29;stroke:#ff7d9b;stroke-width:2}.line{stroke:#68d6ff;stroke-width:3;fill:none;marker-end:url(#arrow)}.tree{stroke:#8292b3;stroke-width:3}.t{fill:#f4f7ff;font:600 23px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}.s{fill:#abb9d2;font:16px -apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif}.m{fill:#dce8ff;font:16px ui-monospace,SFMono-Regular,Menlo,monospace}</style></defs>
+  <rect class="bg" width="1200" height="600" rx="24"/><text class="t" x="55" y="60">Two representations, two sets of invariants</text>
+  <rect class="panel" x="55" y="100" width="515" height="445" rx="20"/><text class="t" x="90" y="145">ft::vector</text><text class="s" x="90" y="177">allocator-owned storage and explicit object lifetime</text>
+  <rect class="live" x="90" y="230" width="105" height="90" rx="12"/><rect class="live" x="205" y="230" width="105" height="90" rx="12"/><rect class="live" x="320" y="230" width="105" height="90" rx="12"/>
+  <rect class="space" x="435" y="230" width="100" height="90" rx="12"/>
+  <text class="m" x="126" y="282">0</text><text class="m" x="241" y="282">1</text><text class="m" x="356" y="282">2</text><text class="m" x="463" y="282">raw</text>
+  <text class="m" x="90" y="360">[data, data + size)</text><text class="s" x="90" y="390">constructed objects</text><text class="m" x="310" y="430">[size, capacity)</text><text class="s" x="310" y="460">uninitialized storage</text>
+  <path class="line" d="M90 500 H520"/><text class="s" x="155" y="525">build new block first, then replace state</text>
+  <rect class="panel" x="630" y="100" width="515" height="445" rx="20"/><text class="t" x="665" y="145">ft::map</text><text class="s" x="665" y="177">header sentinel and red-black tree</text>
+  <rect class="panel" x="790" y="205" width="190" height="75" rx="14"/><text class="m" x="827" y="250">header / end()</text>
+  <circle class="node" cx="885" cy="355" r="45"/><text class="m" x="865" y="361">8 B</text>
+  <circle class="red" cx="760" cy="465" r="40"/><text class="m" x="742" y="471">3 R</text><circle class="red" cx="1010" cy="465" r="40"/><text class="m" x="990" y="471">12 R</text>
+  <line class="tree" x1="855" y1="390" x2="785" y2="435"/><line class="tree" x1="915" y1="390" x2="985" y2="435"/><path class="line" d="M855 280 L870 306"/>
+  <text class="m" x="665" y="235">left → min</text><text class="m" x="1000" y="235">max ← right</text><text class="s" x="668" y="523">black root · no red/red edge · equal black height</text>
+</svg>
