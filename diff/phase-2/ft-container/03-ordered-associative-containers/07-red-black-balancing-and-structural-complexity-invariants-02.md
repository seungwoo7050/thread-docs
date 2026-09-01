## `test(map): 무작위 연산마다 레드-블랙 불변식 검증`

diff --git a/Makefile b/Makefile
index a203ca0..2c07c8f 100644
--- a/Makefile
+++ b/Makefile
@@ -4,7 +4,9 @@ CPPFLAGS := -Iinclude
 
 BUILD_DIR := build
 TEST_NAMES := test_containers test_vector_exceptions test_map_exceptions \
-	test_map_iterators test_map_policy_exceptions
+	test_map_iterators test_map_policy_exceptions test_map_randomized
+
+TEST_SUPPORT_HEADERS := $(wildcard tests/support/*.hpp)
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
 HEADERS := $(wildcard include/*.hpp)
 
@@ -15,7 +17,7 @@ all: $(TEST_BINS)
 $(BUILD_DIR):
 	mkdir -p $(BUILD_DIR)
 
-$(BUILD_DIR)/%: tests/%.cpp $(HEADERS) | $(BUILD_DIR)
+$(BUILD_DIR)/%: tests/%.cpp $(HEADERS) $(TEST_SUPPORT_HEADERS) | $(BUILD_DIR)
 	$(CXX) $(CXXFLAGS) $(CPPFLAGS) $< -o $@
 
 test: $(TEST_BINS)
diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 8c1134f..118530c 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -11,10 +11,19 @@
 
 namespace ft
 {
+	namespace detail
+	{
+		template <class Map>
+		struct map_inspector;
+	}
+
 	template <class Key, class T, class Compare = std::less<Key>,
 		class Alloc = std::allocator<ft::pair<const Key, T> > >
 	class map
 	{
+		template <class Map>
+		friend struct detail::map_inspector;
+
 	public:
 		typedef Key key_type;
 		typedef T mapped_type;
diff --git a/tests/support/map_inspector.hpp b/tests/support/map_inspector.hpp
new file mode 100644
index 0000000..a92e4e5
--- /dev/null
+++ b/tests/support/map_inspector.hpp
@@ -0,0 +1,151 @@
+#ifndef TESTS_SUPPORT_MAP_INSPECTOR_HPP
+# define TESTS_SUPPORT_MAP_INSPECTOR_HPP
+
+# include <algorithm>
+# include <cstddef>
+# include <sstream>
+# include <string>
+# include "ft_map.hpp"
+
+namespace ft
+{
+	namespace detail
+	{
+		struct map_validation
+		{
+			bool valid;
+			std::string message;
+			std::size_t node_count;
+			std::size_t height;
+			std::size_t black_height;
+
+			map_validation()
+				: valid(true), message(), node_count(0), height(0),
+				  black_height(0)
+			{
+			}
+		};
+
+		template <class Key, class T, class Compare, class Alloc>
+		struct map_inspector<ft::map<Key, T, Compare, Alloc> >
+		{
+			typedef ft::map<Key, T, Compare, Alloc> map_type;
+			typedef typename map_type::node_base node_base;
+			typedef typename map_type::key_type key_type;
+
+			static map_validation validate(const map_type& values)
+			{
+				map_validation result;
+				const node_base* header = &values._header;
+				if (!header->is_header)
+					return _fail("header flag is not set");
+				if (header->red)
+					return _fail("header must be black");
+				if (values._size == 0)
+				{
+					if (header->parent != NULL)
+						return _fail("empty header has a root");
+					if (header->left != header || header->right != header)
+						return _fail("empty header extrema do not self-reference");
+					result.black_height = 1;
+					return result;
+				}
+
+				const node_base* root = header->parent;
+				if (root == NULL)
+					return _fail("non-empty map has no root");
+				if (root->parent != header)
+					return _fail("root does not point to header");
+				if (root->red)
+					return _fail("root must be black");
+				if (_minimum(root) != header->left)
+					return _fail("header minimum is stale");
+				if (_maximum(root) != header->right)
+					return _fail("header maximum is stale");
+
+				result = _validate_subtree(values, root, header, NULL, NULL);
+				if (result.valid && result.node_count != values._size)
+				{
+					std::ostringstream message;
+					message << "reachable node count " << result.node_count
+						<< " differs from size " << values._size;
+					return _fail(message.str());
+				}
+				return result;
+			}
+
+			static const key_type& root_key(const map_type& values)
+			{
+				return map_type::_value(values._header.parent).first;
+			}
+
+		private:
+			static map_validation _fail(const std::string& message)
+			{
+				map_validation result;
+				result.valid = false;
+				result.message = message;
+				return result;
+			}
+
+			static const node_base* _minimum(const node_base* current)
+			{
+				while (current->left != NULL)
+					current = current->left;
+				return current;
+			}
+
+			static const node_base* _maximum(const node_base* current)
+			{
+				while (current->right != NULL)
+					current = current->right;
+				return current;
+			}
+
+			static map_validation _validate_subtree(const map_type& values,
+				const node_base* current, const node_base* expected_parent,
+				const key_type* lower, const key_type* upper)
+			{
+				if (current == NULL)
+				{
+					map_validation leaf;
+					leaf.black_height = 1;
+					return leaf;
+				}
+				if (current->is_header)
+					return _fail("header is reachable as a child");
+				if (current->parent != expected_parent)
+					return _fail("child and parent links disagree");
+
+				const key_type& key = map_type::_value(current).first;
+				if (lower != NULL && !values._comp(*lower, key))
+					return _fail("key is not greater than its lower bound");
+				if (upper != NULL && !values._comp(key, *upper))
+					return _fail("key is not less than its upper bound");
+				if (current->red
+					&& ((current->left != NULL && current->left->red)
+						|| (current->right != NULL && current->right->red)))
+					return _fail("red node has a red child");
+
+				map_validation left = _validate_subtree(values, current->left,
+					current, lower, &key);
+				if (!left.valid)
+					return left;
+				map_validation right = _validate_subtree(values, current->right,
+					current, &key, upper);
+				if (!right.valid)
+					return right;
+				if (left.black_height != right.black_height)
+					return _fail("black heights differ");
+
+				map_validation result;
+				result.node_count = left.node_count + right.node_count + 1;
+				result.height = std::max(left.height, right.height) + 1;
+				result.black_height = left.black_height + (current->red ? 0 : 1);
+				return result;
+			}
+		};
+	}
+}
+
+#endif
diff --git a/tests/test_map_randomized.cpp b/tests/test_map_randomized.cpp
new file mode 100644
index 0000000..5bad435
--- /dev/null
+++ b/tests/test_map_randomized.cpp
@@ -0,0 +1,260 @@
+#include <cstdlib>
+#include <iostream>
+#include <map>
+#include <sstream>
+#include <string>
+#include <vector>
+#include "ft_map.hpp"
+#include "support/map_inspector.hpp"
+
+namespace
+{
+	const unsigned int random_seed = 0x5EED1234U;
+	std::vector<std::string> operation_log;
+	std::size_t current_step = 0;
+
+	class generator
+	{
+	public:
+		explicit generator(unsigned int seed) : _state(seed)
+		{
+		}
+
+		unsigned int next()
+		{
+			_state = _state * 1664525U + 1013904223U;
+			return _state;
+		}
+
+		int key()
+		{
+			return static_cast<int>(next() % 129U) - 64;
+		}
+
+	private:
+		unsigned int _state;
+	};
+
+	void fail(const std::string& message)
+	{
+		std::cerr << "FAIL: " << message << "\nseed=" << random_seed
+			<< " step=" << current_step << "\noperation prefix:" << std::endl;
+		for (std::size_t i = 0; i < operation_log.size(); ++i)
+			std::cerr << i << ": " << operation_log[i] << std::endl;
+		std::exit(1);
+	}
+
+	void require(bool condition, const std::string& message)
+	{
+		if (!condition)
+			fail(message);
+	}
+
+	void record(const std::string& operation)
+	{
+		operation_log.push_back(operation);
+		current_step = operation_log.size() - 1;
+	}
+
+	void compare_maps(const ft::map<int, int>& actual,
+		const std::map<int, int>& expected, const std::string& label)
+	{
+		require(actual.size() == expected.size(), label + " size");
+		ft::map<int, int>::const_iterator actual_it = actual.begin();
+		std::map<int, int>::const_iterator expected_it = expected.begin();
+		while (actual_it != actual.end() && expected_it != expected.end())
+		{
+			require(actual_it->first == expected_it->first, label + " key");
+			require(actual_it->second == expected_it->second,
+				label + " mapped value");
+			++actual_it;
+			++expected_it;
+		}
+		require(actual_it == actual.end() && expected_it == expected.end(),
+			label + " end");
+	}
+
+	void validate_map(const ft::map<int, int>& values,
+		const std::string& label)
+	{
+		ft::detail::map_validation validation =
+			ft::detail::map_inspector<ft::map<int, int> >::validate(values);
+		require(validation.valid, label + ": " + validation.message);
+	}
+
+	void compare_query(const ft::map<int, int>& actual,
+		const std::map<int, int>& expected, int key)
+	{
+		ft::map<int, int>::const_iterator fit = actual.find(key);
+		std::map<int, int>::const_iterator sit = expected.find(key);
+		require((fit == actual.end()) == (sit == expected.end()),
+			"find presence");
+		if (sit != expected.end())
+			require(fit->second == sit->second, "find value");
+
+		fit = actual.lower_bound(key);
+		sit = expected.lower_bound(key);
+		require((fit == actual.end()) == (sit == expected.end()),
+			"lower_bound presence");
+		if (sit != expected.end())
+			require(fit->first == sit->first, "lower_bound key");
+
+		fit = actual.upper_bound(key);
+		sit = expected.upper_bound(key);
+		require((fit == actual.end()) == (sit == expected.end()),
+			"upper_bound presence");
+		if (sit != expected.end())
+			require(fit->first == sit->first, "upper_bound key");
+	}
+
+	void erase_at(ft::map<int, int>& actual, std::map<int, int>& expected,
+		std::size_t index)
+	{
+		ft::map<int, int>::iterator fit = actual.begin();
+		std::map<int, int>::iterator sit = expected.begin();
+		while (index-- != 0)
+		{
+			++fit;
+			++sit;
+		}
+		actual.erase(fit);
+		expected.erase(sit);
+	}
+
+	void test_fixed_erasure_sequences()
+	{
+		const int insertion[] = {11, 2, 14, 1, 7, 15, 5, 8, 4, 13, 6, 12};
+		const int erasure[] = {14, 15, 11, 2, 1, 7, 5, 8, 4, 13, 6, 12};
+		ft::map<int, int> actual;
+		std::map<int, int> expected;
+		for (std::size_t i = 0; i < sizeof(insertion) / sizeof(insertion[0]); ++i)
+		{
+			std::ostringstream operation;
+			operation << "fixed insert " << insertion[i];
+			record(operation.str());
+			actual.insert(ft::make_pair(insertion[i], insertion[i] * 10));
+			expected.insert(std::make_pair(insertion[i], insertion[i] * 10));
+			validate_map(actual, "fixed insert invariant");
+		}
+		for (std::size_t i = 0; i < sizeof(erasure) / sizeof(erasure[0]); ++i)
+		{
+			std::ostringstream operation;
+			operation << "fixed erase " << erasure[i];
+			record(operation.str());
+			require(actual.erase(erasure[i]) == expected.erase(erasure[i]),
+				"fixed erase count");
+			compare_maps(actual, expected, "fixed erase result");
+			validate_map(actual, "fixed erase invariant");
+		}
+	}
+
+	void test_repeated_root_erasure()
+	{
+		ft::map<int, int> actual;
+		std::map<int, int> expected;
+		for (int key = 0; key < 96; ++key)
+		{
+			actual.insert(ft::make_pair(key, -key));
+			expected.insert(std::make_pair(key, -key));
+		}
+		while (!actual.empty())
+		{
+			int root =
+				ft::detail::map_inspector<ft::map<int, int> >::root_key(actual);
+			std::ostringstream operation;
+			operation << "erase current root " << root;
+			record(operation.str());
+			actual.erase(root);
+			expected.erase(root);
+			compare_maps(actual, expected, "root erase result");
+			validate_map(actual, "root erase invariant");
+		}
+	}
+
+	void test_randomized_differential()
+	{
+		generator random(random_seed);
+		ft::map<int, int> actual;
+		std::map<int, int> expected;
+		ft::map<int, int> secondary_actual;
+		std::map<int, int> secondary_expected;
+
+		for (std::size_t step = 0; step < 3000; ++step)
+		{
+			const unsigned int operation = random.next() % 10U;
+			const int key = random.key();
+			const int value = static_cast<int>(random.next() % 2001U) - 1000;
+			std::ostringstream description;
+			description << "random op=" << operation << " key=" << key
+				<< " value=" << value;
+			record(description.str());
+
+			if (operation == 0)
+			{
+				ft::pair<ft::map<int, int>::iterator, bool> fr =
+					actual.insert(ft::make_pair(key, value));
+				std::pair<std::map<int, int>::iterator, bool> sr =
+					expected.insert(std::make_pair(key, value));
+				require(fr.second == sr.second, "insert result");
+				require(fr.first->first == sr.first->first, "insert iterator");
+			}
+			else if (operation == 1)
+			{
+				actual[key] = value;
+				expected[key] = value;
+			}
+			else if (operation == 2)
+				require(actual.erase(key) == expected.erase(key),
+					"erase key count");
+			else if (operation == 3 && !expected.empty())
+				erase_at(actual, expected,
+					static_cast<std::size_t>(random.next()) % expected.size());
+			else if (operation == 4)
+				compare_query(actual, expected, key);
+			else if (operation == 5)
+			{
+				ft::map<int, int> copy(actual);
+				std::map<int, int> expected_copy(expected);
+				compare_maps(copy, expected_copy, "copy result");
+				validate_map(copy, "copy invariant");
+			}
+			else if (operation == 6)
+			{
+				ft::map<int, int> assigned;
+				assigned.insert(ft::make_pair(999, 999));
+				assigned = actual;
+				compare_maps(assigned, expected, "assignment result");
+				validate_map(assigned, "assignment invariant");
+			}
+			else if (operation == 7)
+			{
+				secondary_actual[key] = value;
+				secondary_expected[key] = value;
+				actual.swap(secondary_actual);
+				expected.swap(secondary_expected);
+			}
+			else if (operation == 8 && random.next() % 17U == 0)
+			{
+				actual.clear();
+				expected.clear();
+			}
+			else
+				compare_query(actual, expected, key);
+
+			compare_maps(actual, expected, "random primary result");
+			compare_maps(secondary_actual, secondary_expected,
+				"random secondary result");
+			validate_map(actual, "random primary invariant");
+			validate_map(secondary_actual, "random secondary invariant");
+		}
+	}
+}
+
+int main()
+{
+	test_fixed_erasure_sequences();
+	test_repeated_root_erasure();
+	test_randomized_differential();
+	std::cout << "map randomized checks passed" << std::endl;
+	return 0;
+}


## `perf(map): 높이와 비교 횟수 회귀 상한 추가`

diff --git a/Makefile b/Makefile
index 2c07c8f..a618968 100644
--- a/Makefile
+++ b/Makefile
@@ -4,7 +4,8 @@ CPPFLAGS := -Iinclude
 
 BUILD_DIR := build
 TEST_NAMES := test_containers test_vector_exceptions test_map_exceptions \
-	test_map_iterators test_map_policy_exceptions test_map_randomized
+	test_map_iterators test_map_policy_exceptions test_map_randomized \
+	test_complexity
 
 TEST_SUPPORT_HEADERS := $(wildcard tests/support/*.hpp)
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
diff --git a/tests/test_complexity.cpp b/tests/test_complexity.cpp
new file mode 100644
index 0000000..8ff9a5a
--- /dev/null
+++ b/tests/test_complexity.cpp
@@ -0,0 +1,146 @@
+#include <cstdlib>
+#include <iostream>
+#include <sstream>
+#include <string>
+#include <vector>
+#include "ft_map.hpp"
+#include "support/map_inspector.hpp"
+
+namespace
+{
+	class counting_less
+	{
+	public:
+		explicit counting_less(std::size_t* comparisons = NULL)
+			: _comparisons(comparisons)
+		{
+		}
+
+		bool operator()(int lhs, int rhs) const
+		{
+			if (_comparisons != NULL)
+				++(*_comparisons);
+			return lhs < rhs;
+		}
+
+	private:
+		std::size_t* _comparisons;
+	};
+
+	typedef ft::map<int, int, counting_less> measured_map;
+
+	void require(bool condition, const std::string& message)
+	{
+		if (!condition)
+		{
+			std::cerr << "FAIL: " << message << std::endl;
+			std::exit(1);
+		}
+	}
+
+	std::size_t ceil_log2(std::size_t value)
+	{
+		std::size_t exponent = 0;
+		std::size_t power = 1;
+		while (power < value)
+		{
+			power *= 2;
+			++exponent;
+		}
+		return exponent;
+	}
+
+	void make_ascending(std::vector<int>& keys)
+	{
+		for (std::size_t i = 0; i < keys.size(); ++i)
+			keys[i] = static_cast<int>(i);
+	}
+
+	void make_descending(std::vector<int>& keys)
+	{
+		for (std::size_t i = 0; i < keys.size(); ++i)
+			keys[i] = static_cast<int>(keys.size() - i - 1);
+	}
+
+	void make_fixed_random(std::vector<int>& keys)
+	{
+		make_ascending(keys);
+		unsigned int state = 0xC0FFEE11U;
+		for (std::size_t i = keys.size(); i > 1; --i)
+		{
+			state = state * 1664525U + 1013904223U;
+			const std::size_t other = state % i;
+			const int tmp = keys[i - 1];
+			keys[i - 1] = keys[other];
+			keys[other] = tmp;
+		}
+	}
+
+	void check_scenario(const std::string& label,
+		const std::vector<int>& insertion_order)
+	{
+		std::size_t comparisons = 0;
+		measured_map values((counting_less(&comparisons)));
+		for (std::size_t i = 0; i < insertion_order.size(); ++i)
+			values.insert(ft::make_pair(insertion_order[i],
+				insertion_order[i] * 3));
+		const std::size_t insertion_comparisons = comparisons;
+
+		ft::detail::map_validation validation =
+			ft::detail::map_inspector<measured_map>::validate(values);
+		require(validation.valid, label + " invariant: " + validation.message);
+		const std::size_t logarithm = ceil_log2(values.size() + 1);
+		const std::size_t height_limit = 2 * logarithm;
+		require(validation.height <= height_limit,
+			label + " red-black height limit");
+		const std::size_t insertion_limit =
+			values.size() * (4 * logarithm + 4);
+		require(insertion_comparisons <= insertion_limit,
+			label + " insertion comparison limit");
+
+		std::size_t maximum_find_comparisons = 0;
+		for (std::size_t i = 0; i < insertion_order.size(); ++i)
+		{
+			comparisons = 0;
+			measured_map::const_iterator found =
+				values.find(insertion_order[i]);
+			require(found != values.end(), label + " find existing key");
+			if (comparisons > maximum_find_comparisons)
+				maximum_find_comparisons = comparisons;
+			require(comparisons <= 2 * validation.height + 2,
+				label + " find comparison limit");
+		}
+		const int missing_keys[] = {-1,
+			static_cast<int>(insertion_order.size())};
+		for (std::size_t i = 0; i < 2; ++i)
+		{
+			comparisons = 0;
+			require(values.find(missing_keys[i]) == values.end(),
+				label + " find missing key");
+			if (comparisons > maximum_find_comparisons)
+				maximum_find_comparisons = comparisons;
+			require(comparisons <= 2 * validation.height + 2,
+				label + " missing find comparison limit");
+		}
+
+		std::cout << label << ": nodes=" << values.size()
+			<< " height=" << validation.height
+			<< " insert_comparisons=" << insertion_comparisons
+			<< " max_find_comparisons=" << maximum_find_comparisons
+			<< std::endl;
+	}
+}
+
+int main()
+{
+	const std::size_t node_count = 1024;
+	std::vector<int> keys(node_count);
+	make_ascending(keys);
+	check_scenario("ascending", keys);
+	make_descending(keys);
+	check_scenario("descending", keys);
+	make_fixed_random(keys);
+	check_scenario("fixed-random", keys);
+	std::cout << "complexity checks passed" << std::endl;
+	return 0;
+}
