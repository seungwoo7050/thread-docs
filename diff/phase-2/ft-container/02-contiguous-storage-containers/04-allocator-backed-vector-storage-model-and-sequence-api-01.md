# Allocator 기반 벡터 저장 모델과 시퀀스 API

## `feat(vector): allocator 기반 저장소 수명 관리`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
new file mode 100644
index 0000000..011004e
--- /dev/null
+++ b/include/ft_vector.hpp
@@ -0,0 +1,89 @@
+#ifndef FT_VECTOR_HPP
+# define FT_VECTOR_HPP
+
+# include <algorithm>
+# include <cstddef>
+# include <limits>
+# include <memory>
+# include <stdexcept>
+# include "ft_algorithm.hpp"
+# include "ft_iterator.hpp"
+# include "ft_type_traits.hpp"
+
+namespace ft
+{
+	template <class T, class Alloc = std::allocator<T> >
+	class vector
+	{
+	public:
+		typedef T value_type;
+		typedef Alloc allocator_type;
+		typedef typename allocator_type::reference reference;
+		typedef typename allocator_type::const_reference const_reference;
+		typedef typename allocator_type::pointer pointer;
+		typedef typename allocator_type::const_pointer const_pointer;
+		typedef pointer iterator;
+		typedef const_pointer const_iterator;
+		typedef ft::reverse_iterator<iterator> reverse_iterator;
+		typedef ft::reverse_iterator<const_iterator> const_reverse_iterator;
+		typedef std::ptrdiff_t difference_type;
+		typedef std::size_t size_type;
+
+		explicit vector(const allocator_type& alloc = allocator_type())
+			: _alloc(alloc), _data(NULL), _size(0), _capacity(0)
+		{
+		}
+
+		explicit vector(size_type count, const value_type& value = value_type(),
+			const allocator_type& alloc = allocator_type())
+			: _alloc(alloc), _data(NULL), _size(0), _capacity(0)
+		{
+			_assign_fill(count, value);
+		}
+
+		~vector()
+		{
+			_destroy_storage();
+		}
+
+		size_type size() const { return _size; }
+		bool empty() const { return _size == 0; }
+		allocator_type get_allocator() const { return _alloc; }
+
+	private:
+		allocator_type _alloc;
+		pointer _data;
+		size_type _size;
+		size_type _capacity;
+
+		void _assign_fill(size_type count, const value_type& value)
+		{
+			if (count == 0)
+				return;
+			_data = _alloc.allocate(count);
+			_capacity = count;
+			try
+			{
+				for (; _size < count; ++_size)
+					_alloc.construct(_data + _size, value);
+			}
+			catch (...)
+			{
+				_destroy_storage();
+				throw;
+			}
+		}
+
+		void _destroy_storage()
+		{
+			while (_size)
+				_alloc.destroy(_data + --_size);
+			if (_data)
+				_alloc.deallocate(_data, _capacity);
+			_data = NULL;
+			_capacity = 0;
+		}
+	};
+}
+
+#endif


## `feat(vector): 반복자와 원소 접근 경계 구현`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 011004e..bee680f 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -46,8 +46,48 @@ namespace ft
 			_destroy_storage();
 		}
 
+		iterator begin() { return _data; }
+		const_iterator begin() const { return _data; }
+		iterator end() { return _data + _size; }
+		const_iterator end() const { return _data + _size; }
+
+		reverse_iterator rbegin() { return reverse_iterator(end()); }
+		const_reverse_iterator rbegin() const
+		{
+			return const_reverse_iterator(end());
+		}
+
+		reverse_iterator rend() { return reverse_iterator(begin()); }
+		const_reverse_iterator rend() const
+		{
+			return const_reverse_iterator(begin());
+		}
+
 		size_type size() const { return _size; }
 		bool empty() const { return _size == 0; }
+
+		reference operator[](size_type pos) { return _data[pos]; }
+		const_reference operator[](size_type pos) const { return _data[pos]; }
+
+		reference at(size_type pos)
+		{
+			if (pos >= _size)
+				throw std::out_of_range("ft::vector::at");
+			return _data[pos];
+		}
+
+		const_reference at(size_type pos) const
+		{
+			if (pos >= _size)
+				throw std::out_of_range("ft::vector::at");
+			return _data[pos];
+		}
+
+		reference front() { return _data[0]; }
+		const_reference front() const { return _data[0]; }
+		reference back() { return _data[_size - 1]; }
+		const_reference back() const { return _data[_size - 1]; }
+
 		allocator_type get_allocator() const { return _alloc; }
 
 	private:


## `feat(vector): 용량 확장과 원소 재배치 구현`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index bee680f..e5188b7 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -64,7 +64,17 @@ namespace ft
 		}
 
 		size_type size() const { return _size; }
+		size_type capacity() const { return _capacity; }
 		bool empty() const { return _size == 0; }
+		size_type max_size() const { return _alloc.max_size(); }
+
+		void reserve(size_type new_cap)
+		{
+			if (new_cap > max_size())
+				throw std::length_error("ft::vector::reserve");
+			if (new_cap > _capacity)
+				_reallocate(new_cap);
+		}
 
 		reference operator[](size_type pos) { return _data[pos]; }
 		const_reference operator[](size_type pos) const { return _data[pos]; }
@@ -96,6 +106,41 @@ namespace ft
 		size_type _size;
 		size_type _capacity;
 
+		size_type _next_capacity(size_type minimum) const
+		{
+			size_type next = _capacity == 0 ? 1 : _capacity * 2;
+			if (next < minimum)
+				next = minimum;
+			if (next > max_size())
+				throw std::length_error("ft::vector capacity");
+			return next;
+		}
+
+		void _reallocate(size_type new_cap)
+		{
+			pointer new_data = _alloc.allocate(new_cap);
+			size_type i = 0;
+			try
+			{
+				for (; i < _size; ++i)
+					_alloc.construct(new_data + i, _data[i]);
+			}
+			catch (...)
+			{
+				while (i)
+					_alloc.destroy(new_data + --i);
+				_alloc.deallocate(new_data, new_cap);
+				throw;
+			}
+			while (_size)
+				_alloc.destroy(_data + --_size);
+			if (_data)
+				_alloc.deallocate(_data, _capacity);
+			_data = new_data;
+			_size = i;
+			_capacity = new_cap;
+		}
+
 		void _assign_fill(size_type count, const value_type& value)
 		{
 			if (count == 0)


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


## `test(vector): 핵심 공개 동작을 표준 결과와 비교`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 74f016e..89e2482 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -1,11 +1,14 @@
 #include <cstdlib>
 #include <iostream>
+#include <stdexcept>
 #include <string>
+#include <vector>
 
 #include "ft_algorithm.hpp"
 #include "ft_iterator.hpp"
 #include "ft_pair.hpp"
 #include "ft_type_traits.hpp"
+#include "ft_vector.hpp"
 
 namespace
 {
@@ -18,6 +21,16 @@ namespace
 		}
 	}
 
+	template <class FtVector, class StdVector>
+	void compare_vector(const FtVector& ftv, const StdVector& stdv,
+		const std::string& label)
+	{
+		require(ftv.size() == stdv.size(), label + " size");
+		require(ftv.empty() == stdv.empty(), label + " empty");
+		for (std::size_t i = 0; i < stdv.size(); ++i)
+			require(ftv[i] == stdv[i], label + " element");
+	}
+
 	void test_utilities()
 	{
 		ft::pair<int, std::string> p = ft::make_pair(3, std::string("three"));
@@ -36,11 +49,49 @@ namespace
 		require(ft::iterator_traits<int*>::difference_type(3) == 3,
 			"iterator_traits difference type");
 	}
+
+	void test_vector()
+	{
+		ft::vector<int> ftv;
+		std::vector<int> stdv;
+		for (int i = 0; i < 32; ++i)
+		{
+			ftv.push_back(i * 3 - 7);
+			stdv.push_back(i * 3 - 7);
+		}
+		compare_vector(ftv, stdv, "push_back");
+
+		ftv.insert(ftv.begin() + 4, 3, 42);
+		stdv.insert(stdv.begin() + 4, 3, 42);
+		ftv.erase(ftv.begin() + 2, ftv.begin() + 7);
+		stdv.erase(stdv.begin() + 2, stdv.begin() + 7);
+		ftv.resize(40, -9);
+		stdv.resize(40, -9);
+		ftv.reserve(96);
+		stdv.reserve(96);
+		compare_vector(ftv, stdv, "insert erase resize reserve");
+		require(ftv.capacity() >= stdv.size(), "capacity remains usable");
+
+		ft::vector<int> ftcopy(ftv.begin(), ftv.end());
+		std::vector<int> stdcopy(stdv.begin(), stdv.end());
+		compare_vector(ftcopy, stdcopy, "range constructor");
+		require(ftcopy == ftv, "vector equality");
+		require(!(ftcopy < ftv), "vector less equal case");
+
+		bool ft_thrown = false;
+		bool std_thrown = false;
+		try { (void)ftv.at(ftv.size()); }
+		catch (const std::out_of_range&) { ft_thrown = true; }
+		try { (void)stdv.at(stdv.size()); }
+		catch (const std::out_of_range&) { std_thrown = true; }
+		require(ft_thrown == std_thrown, "vector at out_of_range");
+	}
 }
 
 int main()
 {
 	test_utilities();
+	test_vector();
 	std::cout << "ft_containers checks passed" << std::endl;
 	return 0;
 }


## `fix(vector): 용량 계산을 allocator 상한에서 포화`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 1135302..095483e 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -100,6 +100,8 @@ namespace ft
 
 		void resize(size_type count, value_type value = value_type())
 		{
+			if (count > max_size())
+				throw std::length_error("ft::vector::resize");
 			if (count < _size)
 			{
 				while (_size > count)
@@ -136,6 +138,8 @@ namespace ft
 
 		void assign(size_type count, const value_type& value)
 		{
+			if (count > max_size())
+				throw std::length_error("ft::vector::assign");
 			clear();
 			if (count > _capacity)
 			{
@@ -181,6 +185,8 @@ namespace ft
 			size_type index = static_cast<size_type>(pos - begin());
 			if (count == 0)
 				return;
+			if (count > max_size() - _size)
+				throw std::length_error("ft::vector::insert");
 			if (_size + count > _capacity)
 				reserve(_next_capacity(_size + count));
 			for (size_type i = _size; i > index; --i)
@@ -243,11 +249,18 @@ namespace ft
 
 		size_type _next_capacity(size_type minimum) const
 		{
-			size_type next = _capacity == 0 ? 1 : _capacity * 2;
+			const size_type limit = max_size();
+			if (minimum > limit)
+				throw std::length_error("ft::vector capacity");
+			size_type next;
+			if (_capacity == 0)
+				next = 1;
+			else if (_capacity > limit - _capacity)
+				next = limit;
+			else
+				next = _capacity * 2;
 			if (next < minimum)
 				next = minimum;
-			if (next > max_size())
-				throw std::length_error("ft::vector capacity");
 			return next;
 		}
 


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


