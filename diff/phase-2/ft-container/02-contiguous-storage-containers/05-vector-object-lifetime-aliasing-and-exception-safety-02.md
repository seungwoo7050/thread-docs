## `test(vector): 생성·대입·크기 변경 실패 주입`

diff --git a/tests/test_vector_exceptions.cpp b/tests/test_vector_exceptions.cpp
index a1c6b97..6162293 100644
--- a/tests/test_vector_exceptions.cpp
+++ b/tests/test_vector_exceptions.cpp
@@ -2,15 +2,92 @@
 #include <iostream>
 #include <memory>
 #include <new>
+#include <set>
 #include <stdexcept>
 #include <string>
-
 #include "ft_vector.hpp"
 
 namespace
 {
+	class injected_failure : public std::exception
+	{
+	public:
+		const char* what() const throw()
+		{
+			return "injected element failure";
+		}
+	};
+
+	class tracked_value
+	{
+	public:
+		int value;
+
+		static std::set<const void*> live;
+		static int copy_attempts;
+		static int assignment_attempts;
+		static int throw_on_copy;
+		static int throw_on_assignment;
+		static int invalid_copy;
+		static int invalid_destroy;
+
+		explicit tracked_value(int number = 0) : value(number)
+		{
+			live.insert(this);
+		}
+
+		tracked_value(const tracked_value& other)
+		{
+			if (live.find(&other) == live.end())
+			{
+				++invalid_copy;
+				throw injected_failure();
+			}
+			if (copy_attempts++ == throw_on_copy)
+				throw injected_failure();
+			value = other.value;
+			live.insert(this);
+		}
+
+		~tracked_value()
+		{
+			if (live.erase(this) != 1)
+				++invalid_destroy;
+		}
+
+		tracked_value& operator=(const tracked_value& other)
+		{
+			if (live.find(this) == live.end()
+				|| live.find(&other) == live.end())
+			{
+				++invalid_copy;
+				throw injected_failure();
+			}
+			if (assignment_attempts++ == throw_on_assignment)
+				throw injected_failure();
+			value = other.value;
+			return *this;
+		}
+
+		static void reset_failures()
+		{
+			copy_attempts = 0;
+			assignment_attempts = 0;
+			throw_on_copy = -1;
+			throw_on_assignment = -1;
+		}
+	};
+
+	std::set<const void*> tracked_value::live;
+	int tracked_value::copy_attempts = 0;
+	int tracked_value::assignment_attempts = 0;
+	int tracked_value::throw_on_copy = -1;
+	int tracked_value::throw_on_assignment = -1;
+	int tracked_value::invalid_copy = 0;
+	int tracked_value::invalid_destroy = 0;
+
 	template <class T>
-	class bounded_allocator
+	class tracking_allocator
 	{
 	public:
 		typedef T value_type;
@@ -24,18 +101,20 @@ namespace
 		template <class U>
 		struct rebind
 		{
-			typedef bounded_allocator<U> other;
+			typedef tracking_allocator<U> other;
 		};
 
 		static int outstanding_blocks;
+		static int allocation_attempts;
+		static int throw_on_allocate;
 		static size_type size_limit;
 
-		bounded_allocator()
+		tracking_allocator()
 		{
 		}
 
 		template <class U>
-		bounded_allocator(const bounded_allocator<U>&)
+		tracking_allocator(const tracking_allocator<U>&)
 		{
 		}
 
@@ -43,6 +122,8 @@ namespace
 		{
 			if (count > max_size())
 				throw std::bad_alloc();
+			if (allocation_attempts++ == throw_on_allocate)
+				throw std::bad_alloc();
 			pointer result = std::allocator<T>().allocate(count);
 			++outstanding_blocks;
 			return result;
@@ -67,29 +148,46 @@ namespace
 		size_type max_size() const
 		{
 			const size_type normal = std::allocator<T>().max_size();
-			return size_limit < normal ? size_limit : normal;
+			return size_limit != 0 && size_limit < normal ? size_limit : normal;
+		}
+
+		static void reset_allocation_failures()
+		{
+			allocation_attempts = 0;
+			throw_on_allocate = -1;
+			size_limit = 0;
 		}
 	};
 
 	template <class T>
-	int bounded_allocator<T>::outstanding_blocks = 0;
+	int tracking_allocator<T>::outstanding_blocks = 0;
+
+	template <class T>
+	int tracking_allocator<T>::allocation_attempts = 0;
 
 	template <class T>
-	typename bounded_allocator<T>::size_type bounded_allocator<T>::size_limit = 5;
+	int tracking_allocator<T>::throw_on_allocate = -1;
+
+	template <class T>
+	typename tracking_allocator<T>::size_type
+		tracking_allocator<T>::size_limit = 0;
 
 	template <class T, class U>
-	bool operator==(const bounded_allocator<T>&, const bounded_allocator<U>&)
+	bool operator==(const tracking_allocator<T>&, const tracking_allocator<U>&)
 	{
 		return true;
 	}
 
 	template <class T, class U>
-	bool operator!=(const bounded_allocator<T>& lhs,
-		const bounded_allocator<U>& rhs)
+	bool operator!=(const tracking_allocator<T>& lhs,
+		const tracking_allocator<U>& rhs)
 	{
 		return !(lhs == rhs);
 	}
 
+	typedef tracking_allocator<tracked_value> value_allocator;
+	typedef ft::vector<tracked_value, value_allocator> tracked_vector;
+
 	void require(bool condition, const std::string& message)
 	{
 		if (!condition)
@@ -99,10 +197,162 @@ namespace
 		}
 	}
 
-	void test_bounded_growth()
+	void reset_injection()
+	{
+		tracked_value::reset_failures();
+		value_allocator::reset_allocation_failures();
+	}
+
+	void require_clean(const std::string& label)
+	{
+		require(tracked_value::live.empty(), label + " live objects");
+		require(tracked_value::invalid_copy == 0, label + " invalid copy");
+		require(tracked_value::invalid_destroy == 0,
+			label + " invalid destruction");
+		require(value_allocator::outstanding_blocks == 0,
+			label + " allocated blocks");
+	}
+
+	void require_values(const tracked_vector& values, const int* expected,
+		std::size_t count, const std::string& label)
+	{
+		require(values.size() == count, label + " size");
+		for (std::size_t i = 0; i < count; ++i)
+			require(values[i].value == expected[i], label + " value");
+	}
+
+	void test_fill_constructor_rollback()
 	{
-		typedef bounded_allocator<int> int_allocator;
+		reset_injection();
+		{
+			tracked_value seed(7);
+			tracked_value::throw_on_copy = 2;
+			bool thrown = false;
+			try
+			{
+				tracked_vector values(5, seed);
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			require(thrown, "fill constructor injects a failure");
+			require(tracked_value::live.size() == 1,
+				"fill constructor destroys its prefix");
+			require(value_allocator::outstanding_blocks == 0,
+				"fill constructor releases its block");
+		}
+		require_clean("fill constructor rollback");
+	}
+
+	void test_assign_preserves_original()
+	{
+		reset_injection();
+		{
+			tracked_value original(3);
+			tracked_value replacement(9);
+			tracked_vector values(3, original);
+			const int expected[] = {3, 3, 3};
+			tracked_value::copy_attempts = 0;
+			tracked_value::throw_on_copy = 1;
+			bool thrown = false;
+			try
+			{
+				values.assign(5, replacement);
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			require(thrown, "fill assign injects a failure");
+			require_values(values, expected, 3,
+				"fill assign preserves original");
+
+			values.assign(4, values[1]);
+			const int self_expected[] = {3, 3, 3, 3};
+			require_values(values, self_expected, 4,
+				"fill assign snapshots an aliased value");
+		}
+		require_clean("fill assign");
+	}
+
+	void test_copy_assignment_preserves_original()
+	{
+		reset_injection();
+		{
+			tracked_value source_value(5);
+			tracked_value target_value(8);
+			tracked_vector source(5, source_value);
+			tracked_vector target(2, target_value);
+			const int expected[] = {8, 8};
+			tracked_value::copy_attempts = 0;
+			tracked_value::throw_on_copy = 3;
+			bool thrown = false;
+			try
+			{
+				target = source;
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			require(thrown, "copy assignment injects a failure");
+			require_values(target, expected, 2,
+				"copy assignment preserves original");
+		}
+		require_clean("copy assignment");
+	}
+
+	void test_resize_rollback()
+	{
+		reset_injection();
+		{
+			tracked_value original(4);
+			tracked_value appended(6);
+			tracked_vector values(2, original);
+			values.reserve(8);
+			const int expected[] = {4, 4};
+			tracked_value::copy_attempts = 0;
+			tracked_value::throw_on_copy = 1;
+			bool thrown = false;
+			try
+			{
+				values.resize(5, appended);
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			require(thrown, "resize injects a failure");
+			require_values(values, expected, 2, "resize rolls back suffix");
+		}
+		require_clean("resize rollback");
+	}
+
+	void test_aliased_push_back()
+	{
+		reset_injection();
+		{
+			tracked_value seed(11);
+			tracked_vector values(1, seed);
+			values.push_back(values[0]);
+			const int expected[] = {11, 11};
+			require_values(values, expected, 2,
+				"push_back snapshots an aliased value");
+		}
+		require_clean("aliased push_back");
+	}
+
+	void test_bounded_growth_and_empty_iterators()
+	{
+		typedef tracking_allocator<int> int_allocator;
 		typedef ft::vector<int, int_allocator> int_vector;
+		int_allocator::reset_allocation_failures();
+		int_allocator::size_limit = 5;
 		{
 			int_vector values;
 			require(values.begin() == values.end(), "empty begin equals end");
@@ -114,25 +364,21 @@ namespace
 			require(values.size() == 4, "bounded allocator accepts growth");
 			require(values.capacity() == 5,
 				"bounded allocator saturates capacity");
-			bool thrown = false;
-			try
-			{
-				values.reserve(6);
-			}
-			catch (const std::length_error&)
-			{
-				thrown = true;
-			}
-			require(thrown, "bounded allocator rejects excess capacity");
 		}
 		require(int_allocator::outstanding_blocks == 0,
 			"bounded allocator releases storage");
+		int_allocator::reset_allocation_failures();
 	}
 }
 
 int main()
 {
-	test_bounded_growth();
+	test_fill_constructor_rollback();
+	test_assign_preserves_original();
+	test_copy_assignment_preserves_original();
+	test_resize_rollback();
+	test_aliased_push_back();
+	test_bounded_growth_and_empty_iterators();
 	std::cout << "vector exception checks passed" << std::endl;
 	return 0;
 }


## `fix(vector): fill·range 삽입의 객체 수명 보존`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 3977b9e..e804ce1 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -201,16 +201,12 @@ namespace ft
 			size_type index = _index_of(pos);
 			if (count > max_size() - _size)
 				throw std::length_error("ft::vector::insert");
+			value_type value_copy(value);
 			if (_size + count > _capacity)
-				reserve(_next_capacity(_size + count));
-			for (size_type i = _size; i > index; --i)
-			{
-				_alloc.construct(_data + i + count - 1, _data[i - 1]);
-				_alloc.destroy(_data + i - 1);
-			}
-			for (size_type i = 0; i < count; ++i)
-				_alloc.construct(_data + index + i, value);
-			_size += count;
+				_insert_fill_reallocate(index, count, value_copy,
+					_next_capacity(_size + count));
+			else
+				_insert_fill_in_place(index, count, value_copy);
 		}
 
 		template <class InputIt>
@@ -223,13 +219,11 @@ namespace ft
 				return;
 			if (tmp.size() > max_size() - _size)
 				throw std::length_error("ft::vector::insert");
-			vector tail(begin() + index, end(), _alloc);
-			erase(begin() + index, end());
-			reserve(_next_capacity(_size + tmp.size() + tail.size()));
-			for (size_type i = 0; i < tmp.size(); ++i)
-				push_back(tmp[i]);
-			for (size_type i = 0; i < tail.size(); ++i)
-				push_back(tail[i]);
+			if (_size + tmp.size() > _capacity)
+				_insert_range_reallocate(index, tmp,
+					_next_capacity(_size + tmp.size()));
+			else
+				_insert_range_in_place(index, tmp);
 		}
 
 		iterator erase(iterator pos)
@@ -334,6 +328,146 @@ namespace ft
 			std::swap(_capacity, other._capacity);
 		}
 
+		void _replace_storage(pointer new_data, size_type new_size,
+			size_type new_capacity)
+		{
+			_destroy_storage();
+			_data = new_data;
+			_size = new_size;
+			_capacity = new_capacity;
+		}
+
+		void _destroy_constructed_tail(size_type start, size_type count)
+		{
+			while (count)
+				_alloc.destroy(_data + start + --count);
+		}
+
+		void _insert_fill_reallocate(size_type index, size_type count,
+			const value_type& value, size_type new_capacity)
+		{
+			pointer new_data = _alloc.allocate(new_capacity);
+			size_type constructed = 0;
+			try
+			{
+				for (size_type i = 0; i < index; ++i, ++constructed)
+					_alloc.construct(new_data + constructed, _data[i]);
+				for (size_type i = 0; i < count; ++i, ++constructed)
+					_alloc.construct(new_data + constructed, value);
+				for (size_type i = index; i < _size; ++i, ++constructed)
+					_alloc.construct(new_data + constructed, _data[i]);
+			}
+			catch (...)
+			{
+				while (constructed)
+					_alloc.destroy(new_data + --constructed);
+				_alloc.deallocate(new_data, new_capacity);
+				throw;
+			}
+			_replace_storage(new_data, constructed, new_capacity);
+		}
+
+		void _insert_fill_in_place(size_type index, size_type count,
+			const value_type& value)
+		{
+			const size_type old_size = _size;
+			const size_type tail_size = old_size - index;
+			size_type constructed = 0;
+			try
+			{
+				if (count <= tail_size)
+				{
+					for (; constructed < count; ++constructed)
+						_alloc.construct(_data + old_size + constructed,
+							_data[old_size - count + constructed]);
+					for (size_type i = old_size - count; i > index; --i)
+						_data[i + count - 1] = _data[i - 1];
+					for (size_type i = 0; i < count; ++i)
+						_data[index + i] = value;
+				}
+				else
+				{
+					const size_type extra = count - tail_size;
+					for (; constructed < extra; ++constructed)
+						_alloc.construct(_data + old_size + constructed, value);
+					for (size_type i = 0; i < tail_size; ++i, ++constructed)
+						_alloc.construct(_data + old_size + constructed,
+							_data[index + i]);
+					for (size_type i = 0; i < tail_size; ++i)
+						_data[index + i] = value;
+				}
+			}
+			catch (...)
+			{
+				_destroy_constructed_tail(old_size, constructed);
+				throw;
+			}
+			_size = old_size + count;
+		}
+
+		void _insert_range_reallocate(size_type index, const vector& values,
+			size_type new_capacity)
+		{
+			pointer new_data = _alloc.allocate(new_capacity);
+			size_type constructed = 0;
+			try
+			{
+				for (size_type i = 0; i < index; ++i, ++constructed)
+					_alloc.construct(new_data + constructed, _data[i]);
+				for (size_type i = 0; i < values.size(); ++i, ++constructed)
+					_alloc.construct(new_data + constructed, values[i]);
+				for (size_type i = index; i < _size; ++i, ++constructed)
+					_alloc.construct(new_data + constructed, _data[i]);
+			}
+			catch (...)
+			{
+				while (constructed)
+					_alloc.destroy(new_data + --constructed);
+				_alloc.deallocate(new_data, new_capacity);
+				throw;
+			}
+			_replace_storage(new_data, constructed, new_capacity);
+		}
+
+		void _insert_range_in_place(size_type index, const vector& values)
+		{
+			const size_type count = values.size();
+			const size_type old_size = _size;
+			const size_type tail_size = old_size - index;
+			size_type constructed = 0;
+			try
+			{
+				if (count <= tail_size)
+				{
+					for (; constructed < count; ++constructed)
+						_alloc.construct(_data + old_size + constructed,
+							_data[old_size - count + constructed]);
+					for (size_type i = old_size - count; i > index; --i)
+						_data[i + count - 1] = _data[i - 1];
+					for (size_type i = 0; i < count; ++i)
+						_data[index + i] = values[i];
+				}
+				else
+				{
+					const size_type extra = count - tail_size;
+					for (; constructed < extra; ++constructed)
+						_alloc.construct(_data + old_size + constructed,
+							values[tail_size + constructed]);
+					for (size_type i = 0; i < tail_size; ++i, ++constructed)
+						_alloc.construct(_data + old_size + constructed,
+							_data[index + i]);
+					for (size_type i = 0; i < tail_size; ++i)
+						_data[index + i] = values[i];
+				}
+			}
+			catch (...)
+			{
+				_destroy_constructed_tail(old_size, constructed);
+				throw;
+			}
+			_size = old_size + count;
+		}
+
 		void _reallocate(size_type new_cap)
 		{
 			pointer new_data = _alloc.allocate(new_cap);


## `test(vector): 삽입 복사·대입·할당 실패 sweep`

diff --git a/tests/test_vector_exceptions.cpp b/tests/test_vector_exceptions.cpp
index 6162293..7956cab 100644
--- a/tests/test_vector_exceptions.cpp
+++ b/tests/test_vector_exceptions.cpp
@@ -347,6 +347,176 @@ namespace
 		require_clean("aliased push_back");
 	}
 
+	void test_fill_insert_alias_and_failures()
+	{
+		reset_injection();
+		{
+			tracked_vector values;
+			values.reserve(8);
+			for (int i = 1; i <= 3; ++i)
+			{
+				tracked_value value(i);
+				values.push_back(value);
+			}
+			values.insert(values.begin(), 2, values.back());
+			const int expected[] = {3, 3, 1, 2, 3};
+			require_values(values, expected, 5,
+				"fill insert snapshots an aliased value");
+		}
+		require_clean("aliased fill insert");
+
+		reset_injection();
+		{
+			tracked_vector values;
+			values.reserve(8);
+			for (int i = 1; i <= 4; ++i)
+			{
+				tracked_value value(i);
+				values.push_back(value);
+			}
+			tracked_value inserted(9);
+			const int expected[] = {1, 2, 3, 4};
+			tracked_value::copy_attempts = 0;
+			tracked_value::throw_on_copy = 1;
+			bool thrown = false;
+			try
+			{
+				values.insert(values.begin() + 1, 2, inserted);
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			require(thrown, "fill insert copy failure");
+			require_values(values, expected, 4,
+				"fill insert restores constructed tail");
+		}
+		require_clean("fill insert copy rollback");
+
+		reset_injection();
+		{
+			tracked_vector values;
+			values.reserve(8);
+			for (int i = 1; i <= 4; ++i)
+			{
+				tracked_value value(i);
+				values.push_back(value);
+			}
+			tracked_value inserted(9);
+			tracked_value::assignment_attempts = 0;
+			tracked_value::throw_on_assignment = 2;
+			bool thrown = false;
+			try
+			{
+				values.insert(values.begin() + 1, 2, inserted);
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			require(thrown, "fill insert assignment failure");
+			require(values.size() == 4,
+				"fill insert failure keeps tracked size");
+			values.push_back(inserted);
+			require(values.size() == 5,
+				"fill insert remains usable after assignment failure");
+		}
+		require_clean("fill insert assignment rollback");
+	}
+
+	void test_range_insert_capacity_and_rollback()
+	{
+		ft::vector<int> values;
+		values.reserve(16);
+		values.push_back(1);
+		values.push_back(2);
+		values.push_back(3);
+		int additions[] = {8, 9};
+		int* first_address = &values[0];
+		values.insert(values.begin() + 1, additions, additions + 2);
+		const int expected[] = {1, 8, 9, 2, 3};
+		require(values.capacity() == 16,
+			"range insert preserves spare capacity");
+		require(&values[0] == first_address,
+			"range insert preserves prefix address");
+		for (std::size_t i = 0; i < 5; ++i)
+			require(values[i] == expected[i], "range insert value");
+
+		for (int fail_at = 0; fail_at < 14; ++fail_at)
+		{
+			reset_injection();
+			bool thrown = false;
+			{
+				tracked_value one(1);
+				tracked_value two(2);
+				tracked_value seven(7);
+				tracked_value eight(8);
+				tracked_vector original;
+				original.push_back(one);
+				original.push_back(two);
+				tracked_value input[] = {seven, eight};
+				tracked_value::copy_attempts = 0;
+				tracked_value::throw_on_copy = fail_at;
+				try
+				{
+					original.insert(original.begin() + 1, input, input + 2);
+				}
+				catch (const injected_failure&)
+				{
+					thrown = true;
+				}
+				reset_injection();
+				if (thrown)
+				{
+					const int unchanged[] = {1, 2};
+					require_values(original, unchanged, 2,
+						"range insert preserves original");
+				}
+				else
+				{
+					const int inserted[] = {1, 7, 8, 2};
+					require_values(original, inserted, 4,
+						"range insert completes");
+				}
+			}
+			require_clean("range insert failure sweep");
+		}
+	}
+
+	void test_range_insert_allocation_failure()
+	{
+		reset_injection();
+		{
+			tracked_value one(1);
+			tracked_value two(2);
+			tracked_value seven(7);
+			tracked_value eight(8);
+			tracked_vector original;
+			original.push_back(one);
+			original.push_back(two);
+			tracked_value input[] = {seven, eight};
+			value_allocator::allocation_attempts = 0;
+			value_allocator::throw_on_allocate = 2;
+			bool thrown = false;
+			try
+			{
+				original.insert(original.begin() + 1, input, input + 2);
+			}
+			catch (const std::bad_alloc&)
+			{
+				thrown = true;
+			}
+			reset_injection();
+			const int expected[] = {1, 2};
+			require(thrown, "range insert allocation failure");
+			require_values(original, expected, 2,
+				"range insert allocation preserves original");
+		}
+		require_clean("range insert allocation rollback");
+	}
+
 	void test_bounded_growth_and_empty_iterators()
 	{
 		typedef tracking_allocator<int> int_allocator;
@@ -378,6 +548,9 @@ int main()
 	test_copy_assignment_preserves_original();
 	test_resize_rollback();
 	test_aliased_push_back();
+	test_fill_insert_alias_and_failures();
+	test_range_insert_capacity_and_rollback();
+	test_range_insert_allocation_failure();
 	test_bounded_growth_and_empty_iterators();
 	std::cout << "vector exception checks passed" << std::endl;
 	return 0;


