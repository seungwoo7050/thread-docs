## `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증`

diff --git a/Makefile b/Makefile
index d9f67ec..a203ca0 100644
--- a/Makefile
+++ b/Makefile
@@ -4,7 +4,7 @@ CPPFLAGS := -Iinclude
 
 BUILD_DIR := build
 TEST_NAMES := test_containers test_vector_exceptions test_map_exceptions \
-	test_map_iterators
+	test_map_iterators test_map_policy_exceptions
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
 HEADERS := $(wildcard include/*.hpp)
 
diff --git a/tests/test_map_policy_exceptions.cpp b/tests/test_map_policy_exceptions.cpp
new file mode 100644
index 0000000..96e8ad1
--- /dev/null
+++ b/tests/test_map_policy_exceptions.cpp
@@ -0,0 +1,294 @@
+#include <cstdlib>
+#include <iostream>
+#include <memory>
+#include <new>
+#include <stdexcept>
+#include <string>
+#include "ft_map.hpp"
+
+namespace
+{
+	class policy_assignment_failure : public std::exception
+	{
+	public:
+		const char* what() const throw()
+		{
+			return "injected comparator assignment failure";
+		}
+	};
+
+	struct comparison_control
+	{
+		int assignments;
+		bool throw_on_assignment;
+
+		comparison_control() : assignments(0), throw_on_assignment(false)
+		{
+		}
+	};
+
+	class throwing_compare
+	{
+	public:
+		explicit throwing_compare(comparison_control* control = NULL,
+			bool reverse = false) : _control(control), _reverse(reverse)
+		{
+		}
+
+		throwing_compare(const throwing_compare& other)
+			: _control(other._control), _reverse(other._reverse)
+		{
+		}
+
+		throwing_compare& operator=(const throwing_compare& other)
+		{
+			if (_control)
+			{
+				++_control->assignments;
+				if (_control->throw_on_assignment)
+					throw policy_assignment_failure();
+			}
+			_control = other._control;
+			_reverse = other._reverse;
+			return *this;
+		}
+
+		bool operator()(int lhs, int rhs) const
+		{
+			return _reverse ? lhs > rhs : lhs < rhs;
+		}
+
+	private:
+		comparison_control* _control;
+		bool _reverse;
+	};
+
+	struct allocation_control
+	{
+		int outstanding_blocks;
+
+		allocation_control() : outstanding_blocks(0)
+		{
+		}
+	};
+
+	template <class T>
+	class tracking_allocator
+	{
+	public:
+		typedef T value_type;
+		typedef T* pointer;
+		typedef const T* const_pointer;
+		typedef T& reference;
+		typedef const T& const_reference;
+		typedef std::size_t size_type;
+		typedef std::ptrdiff_t difference_type;
+
+		template <class U>
+		struct rebind
+		{
+			typedef tracking_allocator<U> other;
+		};
+
+		explicit tracking_allocator(allocation_control* control = NULL)
+			: _control(control)
+		{
+		}
+
+		template <class U>
+		tracking_allocator(const tracking_allocator<U>& other)
+			: _control(other.control())
+		{
+		}
+
+		pointer allocate(size_type count, const void* = 0)
+		{
+			pointer result = std::allocator<T>().allocate(count);
+			if (_control)
+				++_control->outstanding_blocks;
+			return result;
+		}
+
+		void deallocate(pointer data, size_type count)
+		{
+			std::allocator<T>().deallocate(data, count);
+			if (_control)
+				--_control->outstanding_blocks;
+		}
+
+		void construct(pointer place, const_reference value)
+		{
+			::new (static_cast<void*>(place)) T(value);
+		}
+
+		void destroy(pointer place)
+		{
+			place->~T();
+		}
+
+		size_type max_size() const
+		{
+			return std::allocator<T>().max_size();
+		}
+
+		allocation_control* control() const
+		{
+			return _control;
+		}
+
+	private:
+		allocation_control* _control;
+	};
+
+	template <class T, class U>
+	bool operator==(const tracking_allocator<T>& lhs,
+		const tracking_allocator<U>& rhs)
+	{
+		return lhs.control() == rhs.control();
+	}
+
+	template <class T, class U>
+	bool operator!=(const tracking_allocator<T>& lhs,
+		const tracking_allocator<U>& rhs)
+	{
+		return !(lhs == rhs);
+	}
+
+	typedef ft::pair<const int, int> map_value;
+	typedef tracking_allocator<map_value> map_allocator;
+	typedef ft::map<int, int, throwing_compare, map_allocator> tested_map;
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
+	void require_keys(const tested_map& values, const int* expected,
+		std::size_t count, const std::string& label)
+	{
+		require(values.size() == count, label + " size");
+		tested_map::const_iterator it = values.begin();
+		for (std::size_t index = 0; index < count; ++index, ++it)
+		{
+			require(it != values.end(), label + " early end");
+			require(it->first == expected[index], label + " key");
+		}
+		require(it == values.end(), label + " late end");
+	}
+
+	void test_copy_assignment_keeps_target_ownership()
+	{
+		comparison_control source_comparison;
+		comparison_control target_comparison;
+		allocation_control source_allocation;
+		allocation_control target_allocation;
+		{
+			tested_map source((throwing_compare(&source_comparison)),
+				map_allocator(&source_allocation));
+			source.insert(ft::make_pair(1, 10));
+			source.insert(ft::make_pair(2, 20));
+			source.insert(ft::make_pair(3, 30));
+
+			tested_map target((throwing_compare(&target_comparison, true)),
+				map_allocator(&target_allocation));
+			target.insert(ft::make_pair(9, 90));
+			target.insert(ft::make_pair(7, 70));
+			const int expected[] = {9, 7};
+			const int baseline_blocks = target_allocation.outstanding_blocks;
+
+			target_comparison.throw_on_assignment = true;
+			bool thrown = false;
+			try
+			{
+				target = source;
+			}
+			catch (const policy_assignment_failure&)
+			{
+				thrown = true;
+			}
+			target_comparison.throw_on_assignment = false;
+
+			require(thrown, "copy assignment injects comparator failure");
+			require(target_comparison.assignments != 0,
+				"copy assignment reaches comparator exchange");
+			require_keys(target, expected, 2,
+				"failed copy assignment preserves target");
+			require(target_allocation.outstanding_blocks == baseline_blocks,
+				"failed copy assignment releases temporary tree");
+			target.insert(ft::make_pair(8, 80));
+			require(target.find(8) != target.end(),
+				"target remains usable after comparator failure");
+		}
+		require(source_allocation.outstanding_blocks == 0,
+			"source releases all nodes");
+		require(target_allocation.outstanding_blocks == 0,
+			"target releases all nodes");
+	}
+
+	void test_public_swap_keeps_both_owners()
+	{
+		comparison_control left_comparison;
+		comparison_control right_comparison;
+		allocation_control left_allocation;
+		allocation_control right_allocation;
+		{
+			tested_map left((throwing_compare(&left_comparison)),
+				map_allocator(&left_allocation));
+			left.insert(ft::make_pair(1, 10));
+			left.insert(ft::make_pair(2, 20));
+			tested_map right((throwing_compare(&right_comparison, true)),
+				map_allocator(&right_allocation));
+			right.insert(ft::make_pair(9, 90));
+			right.insert(ft::make_pair(7, 70));
+			right.insert(ft::make_pair(5, 50));
+			const int left_expected[] = {1, 2};
+			const int right_expected[] = {9, 7, 5};
+			const int left_blocks = left_allocation.outstanding_blocks;
+			const int right_blocks = right_allocation.outstanding_blocks;
+
+			left_comparison.throw_on_assignment = true;
+			bool thrown = false;
+			try
+			{
+				left.swap(right);
+			}
+			catch (const policy_assignment_failure&)
+			{
+				thrown = true;
+			}
+			left_comparison.throw_on_assignment = false;
+
+			require(thrown, "public swap injects comparator failure");
+			require(left_comparison.assignments != 0,
+				"public swap reaches comparator exchange");
+			require_keys(left, left_expected, 2,
+				"failed public swap preserves left tree");
+			require_keys(right, right_expected, 3,
+				"failed public swap preserves right tree");
+			require(left.get_allocator().control() == &left_allocation,
+				"failed public swap preserves left allocator");
+			require(right.get_allocator().control() == &right_allocation,
+				"failed public swap preserves right allocator");
+			require(left_allocation.outstanding_blocks == left_blocks,
+				"failed public swap preserves left ownership");
+			require(right_allocation.outstanding_blocks == right_blocks,
+				"failed public swap preserves right ownership");
+		}
+		require(left_allocation.outstanding_blocks == 0,
+			"left map releases all nodes");
+		require(right_allocation.outstanding_blocks == 0,
+			"right map releases all nodes");
+	}
+}
+
+int main()
+{
+	test_copy_assignment_keeps_target_ownership();
+	test_public_swap_keeps_both_owners();
+	std::cout << "map policy exception checks passed" << std::endl;
+	return 0;
+}
