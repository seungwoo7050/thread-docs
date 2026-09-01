# 벡터 객체 수명·별칭·예외 안전성

## `feat(vector): 크기 변경과 값 범위 할당 구현`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index e5188b7..4fdaef9 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -38,7 +38,22 @@ namespace ft
 			const allocator_type& alloc = allocator_type())
 			: _alloc(alloc), _data(NULL), _size(0), _capacity(0)
 		{
-			_assign_fill(count, value);
+			assign(count, value);
+		}
+
+		template <class InputIt>
+		vector(InputIt first, InputIt last,
+			const allocator_type& alloc = allocator_type(),
+			typename ft::enable_if<!ft::is_integral<InputIt>::value>::type* = 0)
+			: _alloc(alloc), _data(NULL), _size(0), _capacity(0)
+		{
+			assign(first, last);
+		}
+
+		vector(const vector& other)
+			: _alloc(other._alloc), _data(NULL), _size(0), _capacity(0)
+		{
+			assign(other.begin(), other.end());
 		}
 
 		~vector()
@@ -46,6 +61,13 @@ namespace ft
 			_destroy_storage();
 		}
 
+		vector& operator=(const vector& other)
+		{
+			if (this != &other)
+				assign(other.begin(), other.end());
+			return *this;
+		}
+
 		iterator begin() { return _data; }
 		const_iterator begin() const { return _data; }
 		iterator end() { return _data + _size; }
@@ -76,6 +98,20 @@ namespace ft
 				_reallocate(new_cap);
 		}
 
+		void resize(size_type count, value_type value = value_type())
+		{
+			if (count < _size)
+			{
+				while (_size > count)
+					_alloc.destroy(_data + --_size);
+				return;
+			}
+			if (count > _capacity)
+				reserve(_next_capacity(count));
+			while (_size < count)
+				_alloc.construct(_data + _size++, value);
+		}
+
 		reference operator[](size_type pos) { return _data[pos]; }
 		const_reference operator[](size_type pos) const { return _data[pos]; }
 
@@ -98,6 +134,47 @@ namespace ft
 		reference back() { return _data[_size - 1]; }
 		const_reference back() const { return _data[_size - 1]; }
 
+		void assign(size_type count, const value_type& value)
+		{
+			clear();
+			if (count > _capacity)
+			{
+				_destroy_storage();
+				_data = _alloc.allocate(count);
+				_capacity = count;
+			}
+			for (size_type i = 0; i < count; ++i)
+				_alloc.construct(_data + i, value);
+			_size = count;
+		}
+
+		template <class InputIt>
+		void assign(InputIt first, InputIt last,
+			typename ft::enable_if<!ft::is_integral<InputIt>::value>::type* = 0)
+		{
+			clear();
+			for (; first != last; ++first)
+				push_back(*first);
+		}
+
+		void push_back(const value_type& value)
+		{
+			if (_size == _capacity)
+				reserve(_next_capacity(_size + 1));
+			_alloc.construct(_data + _size++, value);
+		}
+
+		void pop_back()
+		{
+			_alloc.destroy(_data + --_size);
+		}
+
+		void clear()
+		{
+			while (_size)
+				_alloc.destroy(_data + --_size);
+		}
+
 		allocator_type get_allocator() const { return _alloc; }
 
 	private:
@@ -132,8 +209,7 @@ namespace ft
 				_alloc.deallocate(new_data, new_cap);
 				throw;
 			}
-			while (_size)
-				_alloc.destroy(_data + --_size);
+			clear();
 			if (_data)
 				_alloc.deallocate(_data, _capacity);
 			_data = new_data;
@@ -141,28 +217,9 @@ namespace ft
 			_capacity = new_cap;
 		}
 
-		void _assign_fill(size_type count, const value_type& value)
-		{
-			if (count == 0)
-				return;
-			_data = _alloc.allocate(count);
-			_capacity = count;
-			try
-			{
-				for (; _size < count; ++_size)
-					_alloc.construct(_data + _size, value);
-			}
-			catch (...)
-			{
-				_destroy_storage();
-				throw;
-			}
-		}
-
 		void _destroy_storage()
 		{
-			while (_size)
-				_alloc.destroy(_data + --_size);
+			clear();
 			if (_data)
 				_alloc.deallocate(_data, _capacity);
 			_data = NULL;


## `feat(vector): 중간 변경 연산과 관계 비교 완성`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 4fdaef9..1135302 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -169,12 +169,70 @@ namespace ft
 			_alloc.destroy(_data + --_size);
 		}
 
+		iterator insert(iterator pos, const value_type& value)
+		{
+			size_type index = static_cast<size_type>(pos - begin());
+			insert(pos, 1, value);
+			return begin() + index;
+		}
+
+		void insert(iterator pos, size_type count, const value_type& value)
+		{
+			size_type index = static_cast<size_type>(pos - begin());
+			if (count == 0)
+				return;
+			if (_size + count > _capacity)
+				reserve(_next_capacity(_size + count));
+			for (size_type i = _size; i > index; --i)
+			{
+				_alloc.construct(_data + i + count - 1, _data[i - 1]);
+				_alloc.destroy(_data + i - 1);
+			}
+			for (size_type i = 0; i < count; ++i)
+				_alloc.construct(_data + index + i, value);
+			_size += count;
+		}
+
+		template <class InputIt>
+		void insert(iterator pos, InputIt first, InputIt last,
+			typename ft::enable_if<!ft::is_integral<InputIt>::value>::type* = 0)
+		{
+			size_type index = static_cast<size_type>(pos - begin());
+			for (; first != last; ++first, ++index)
+				insert(begin() + index, *first);
+		}
+
+		iterator erase(iterator pos)
+		{
+			return erase(pos, pos + 1);
+		}
+
+		iterator erase(iterator first, iterator last)
+		{
+			size_type index = static_cast<size_type>(first - begin());
+			size_type count = static_cast<size_type>(last - first);
+			for (size_type i = index; i + count < _size; ++i)
+				_data[i] = _data[i + count];
+			for (size_type i = 0; i < count; ++i)
+				_alloc.destroy(_data + _size - 1 - i);
+			_size -= count;
+			return begin() + index;
+		}
+
 		void clear()
 		{
 			while (_size)
 				_alloc.destroy(_data + --_size);
 		}
 
+		void swap(vector& other)
+		{
+			std::swap(_alloc, other._alloc);
+			std::swap(_data, other._data);
+			std::swap(_size, other._size);
+			std::swap(_capacity, other._capacity);
+		}
+
 		allocator_type get_allocator() const { return _alloc; }
 
 	private:
@@ -226,6 +284,50 @@ namespace ft
 			_capacity = 0;
 		}
 	};
+
+	template <class T, class Alloc>
+	bool operator==(const vector<T, Alloc>& lhs, const vector<T, Alloc>& rhs)
+	{
+		return lhs.size() == rhs.size()
+			&& ft::equal(lhs.begin(), lhs.end(), rhs.begin());
+	}
+
+	template <class T, class Alloc>
+	bool operator!=(const vector<T, Alloc>& lhs, const vector<T, Alloc>& rhs)
+	{
+		return !(lhs == rhs);
+	}
+
+	template <class T, class Alloc>
+	bool operator<(const vector<T, Alloc>& lhs, const vector<T, Alloc>& rhs)
+	{
+		return ft::lexicographical_compare(lhs.begin(), lhs.end(),
+			rhs.begin(), rhs.end());
+	}
+
+	template <class T, class Alloc>
+	bool operator<=(const vector<T, Alloc>& lhs, const vector<T, Alloc>& rhs)
+	{
+		return !(rhs < lhs);
+	}
+
+	template <class T, class Alloc>
+	bool operator>(const vector<T, Alloc>& lhs, const vector<T, Alloc>& rhs)
+	{
+		return rhs < lhs;
+	}
+
+	template <class T, class Alloc>
+	bool operator>=(const vector<T, Alloc>& lhs, const vector<T, Alloc>& rhs)
+	{
+		return !(lhs < rhs);
+	}
+
+	template <class T, class Alloc>
+	void swap(vector<T, Alloc>& lhs, vector<T, Alloc>& rhs)
+	{
+		lhs.swap(rhs);
+	}
 }
 
 #endif


## `test(vector): 제한 allocator에서 용량 상한 검증`

diff --git a/Makefile b/Makefile
index ea63678..9cff901 100644
--- a/Makefile
+++ b/Makefile
@@ -3,7 +3,7 @@ CXXFLAGS := -Wall -Wextra -Werror -std=c++98
 CPPFLAGS := -Iinclude
 
 BUILD_DIR := build
-TEST_NAMES := test_containers
+TEST_NAMES := test_containers test_vector_exceptions
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
 HEADERS := $(wildcard include/*.hpp)
 
diff --git a/tests/test_vector_exceptions.cpp b/tests/test_vector_exceptions.cpp
new file mode 100644
index 0000000..ef31479
--- /dev/null
+++ b/tests/test_vector_exceptions.cpp
@@ -0,0 +1,135 @@
+#include <cstdlib>
+#include <iostream>
+#include <memory>
+#include <new>
+#include <stdexcept>
+#include <string>
+
+#include "ft_vector.hpp"
+
+namespace
+{
+	template <class T>
+	class bounded_allocator
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
+			typedef bounded_allocator<U> other;
+		};
+
+		static int outstanding_blocks;
+		static size_type size_limit;
+
+		bounded_allocator()
+		{
+		}
+
+		template <class U>
+		bounded_allocator(const bounded_allocator<U>&)
+		{
+		}
+
+		pointer allocate(size_type count, const void* = 0)
+		{
+			if (count > max_size())
+				throw std::bad_alloc();
+			pointer result = std::allocator<T>().allocate(count);
+			++outstanding_blocks;
+			return result;
+		}
+
+		void deallocate(pointer data, size_type count)
+		{
+			std::allocator<T>().deallocate(data, count);
+			--outstanding_blocks;
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
+			const size_type normal = std::allocator<T>().max_size();
+			return size_limit < normal ? size_limit : normal;
+		}
+	};
+
+	template <class T>
+	int bounded_allocator<T>::outstanding_blocks = 0;
+
+	template <class T>
+	typename bounded_allocator<T>::size_type bounded_allocator<T>::size_limit = 5;
+
+	template <class T, class U>
+	bool operator==(const bounded_allocator<T>&, const bounded_allocator<U>&)
+	{
+		return true;
+	}
+
+	template <class T, class U>
+	bool operator!=(const bounded_allocator<T>& lhs,
+		const bounded_allocator<U>& rhs)
+	{
+		return !(lhs == rhs);
+	}
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
+	void test_bounded_growth()
+	{
+		typedef bounded_allocator<int> int_allocator;
+		typedef ft::vector<int, int_allocator> int_vector;
+		{
+			int_vector values;
+			values.reserve(3);
+			for (int i = 0; i < 4; ++i)
+				values.push_back(i);
+			require(values.size() == 4, "bounded allocator accepts growth");
+			require(values.capacity() == 5,
+				"bounded allocator saturates capacity");
+			bool thrown = false;
+			try
+			{
+				values.reserve(6);
+			}
+			catch (const std::length_error&)
+			{
+				thrown = true;
+			}
+			require(thrown, "bounded allocator rejects excess capacity");
+		}
+		require(int_allocator::outstanding_blocks == 0,
+			"bounded allocator releases storage");
+	}
+}
+
+int main()
+{
+	test_bounded_growth();
+	std::cout << "vector exception checks passed" << std::endl;
+	return 0;
+}


## `fix(vector): 자기 범위 assign과 insert 입력 보존`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 095483e..6a869a5 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -156,9 +156,10 @@ namespace ft
 		void assign(InputIt first, InputIt last,
 			typename ft::enable_if<!ft::is_integral<InputIt>::value>::type* = 0)
 		{
-			clear();
+			vector tmp(_alloc);
 			for (; first != last; ++first)
-				push_back(*first);
+				tmp.push_back(*first);
+			swap(tmp);
 		}
 
 		void push_back(const value_type& value)
@@ -204,8 +205,18 @@ namespace ft
 			typename ft::enable_if<!ft::is_integral<InputIt>::value>::type* = 0)
 		{
 			size_type index = static_cast<size_type>(pos - begin());
-			for (; first != last; ++first, ++index)
-				insert(begin() + index, *first);
+			vector tmp(first, last, _alloc);
+			if (tmp.empty())
+				return;
+			if (tmp.size() > max_size() - _size)
+				throw std::length_error("ft::vector::insert");
+			vector tail(begin() + index, end(), _alloc);
+			erase(begin() + index, end());
+			reserve(_next_capacity(_size + tmp.size() + tail.size()));
+			for (size_type i = 0; i < tmp.size(); ++i)
+				push_back(tmp[i]);
+			for (size_type i = 0; i < tail.size(); ++i)
+				push_back(tail[i]);
 		}
 
 		iterator erase(iterator pos)


## `test(vector): 자기 범위 변경 결과 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index dfb2319..57d261e 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -77,6 +77,17 @@ namespace
 		require(ftcopy == ftv, "vector equality");
 		require(!(ftcopy < ftv), "vector less equal case");
 
+		std::vector<int> insert_source(stdcopy.begin(), stdcopy.begin() + 4);
+		ftcopy.insert(ftcopy.begin() + 3, ftcopy.begin(), ftcopy.begin() + 4);
+		stdcopy.insert(stdcopy.begin() + 3,
+			insert_source.begin(), insert_source.end());
+		compare_vector(ftcopy, stdcopy, "self range insert");
+
+		std::vector<int> assign_source(stdcopy.begin() + 2, stdcopy.end() - 1);
+		ftcopy.assign(ftcopy.begin() + 2, ftcopy.end() - 1);
+		stdcopy.assign(assign_source.begin(), assign_source.end());
+		compare_vector(ftcopy, stdcopy, "self range assign");
+
 		bool ft_thrown = false;
 		bool std_thrown = false;
 		try { (void)ftv.at(ftv.size()); }


## `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 216aded..3977b9e 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -38,7 +38,9 @@ namespace ft
 			const allocator_type& alloc = allocator_type())
 			: _alloc(alloc), _data(NULL), _size(0), _capacity(0)
 		{
-			assign(count, value);
+			if (count > max_size())
+				throw std::length_error("ft::vector::vector");
+			_initialize_fill(count, value);
 		}
 
 		template <class InputIt>
@@ -110,8 +112,21 @@ namespace ft
 			}
 			if (count > _capacity)
 				reserve(_next_capacity(count));
-			while (_size < count)
-				_alloc.construct(_data + _size++, value);
+			const size_type old_size = _size;
+			try
+			{
+				while (_size < count)
+				{
+					_alloc.construct(_data + _size, value);
+					++_size;
+				}
+			}
+			catch (...)
+			{
+				while (_size > old_size)
+					_alloc.destroy(_data + --_size);
+				throw;
+			}
 		}
 
 		reference operator[](size_type pos) { return _data[pos]; }
@@ -140,16 +155,8 @@ namespace ft
 		{
 			if (count > max_size())
 				throw std::length_error("ft::vector::assign");
-			clear();
-			if (count > _capacity)
-			{
-				_destroy_storage();
-				_data = _alloc.allocate(count);
-				_capacity = count;
-			}
-			for (size_type i = 0; i < count; ++i)
-				_alloc.construct(_data + i, value);
-			_size = count;
+			vector tmp(count, value, _alloc);
+			_swap_storage(tmp);
 		}
 
 		template <class InputIt>
@@ -159,14 +166,20 @@ namespace ft
 			vector tmp(_alloc);
 			for (; first != last; ++first)
 				tmp.push_back(*first);
-			swap(tmp);
+			_swap_storage(tmp);
 		}
 
 		void push_back(const value_type& value)
 		{
 			if (_size == _capacity)
+			{
+				value_type value_copy(value);
 				reserve(_next_capacity(_size + 1));
-			_alloc.construct(_data + _size++, value);
+				_alloc.construct(_data + _size, value_copy);
+			}
+			else
+				_alloc.construct(_data + _size, value);
+			++_size;
 		}
 
 		void pop_back()
@@ -291,6 +304,36 @@ namespace ft
 			return _data ? static_cast<size_type>(pos - _data) : 0;
 		}
 
+		void _initialize_fill(size_type count, const value_type& value)
+		{
+			if (count == 0)
+				return;
+			pointer new_data = _alloc.allocate(count);
+			size_type constructed = 0;
+			try
+			{
+				for (; constructed < count; ++constructed)
+					_alloc.construct(new_data + constructed, value);
+			}
+			catch (...)
+			{
+				while (constructed)
+					_alloc.destroy(new_data + --constructed);
+				_alloc.deallocate(new_data, count);
+				throw;
+			}
+			_data = new_data;
+			_size = count;
+			_capacity = count;
+		}
+
+		void _swap_storage(vector& other)
+		{
+			std::swap(_data, other._data);
+			std::swap(_size, other._size);
+			std::swap(_capacity, other._capacity);
+		}
+
 		void _reallocate(size_type new_cap)
 		{
 			pointer new_data = _alloc.allocate(new_cap);


