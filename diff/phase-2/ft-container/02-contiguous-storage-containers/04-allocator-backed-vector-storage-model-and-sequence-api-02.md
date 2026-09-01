## `test(vector): 역방향 순회 결과 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 57d261e..16451a6 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -88,6 +88,11 @@ namespace
 		stdcopy.assign(assign_source.begin(), assign_source.end());
 		compare_vector(ftcopy, stdcopy, "self range assign");
 
+		ft::vector<int>::reverse_iterator frit = ftcopy.rbegin();
+		std::vector<int>::reverse_iterator srit = stdcopy.rbegin();
+		for (; frit != ftcopy.rend() && srit != stdcopy.rend(); ++frit, ++srit)
+			require(*frit == *srit, "vector reverse iteration");
+
 		bool ft_thrown = false;
 		bool std_thrown = false;
 		try { (void)ftv.at(ftv.size()); }


## `fix(vector): allocator 형식과 빈 반복자 연산 보정`

diff --git a/include/ft_vector.hpp b/include/ft_vector.hpp
index 6a869a5..216aded 100644
--- a/include/ft_vector.hpp
+++ b/include/ft_vector.hpp
@@ -26,8 +26,8 @@ namespace ft
 		typedef const_pointer const_iterator;
 		typedef ft::reverse_iterator<iterator> reverse_iterator;
 		typedef ft::reverse_iterator<const_iterator> const_reverse_iterator;
-		typedef std::ptrdiff_t difference_type;
-		typedef std::size_t size_type;
+		typedef typename allocator_type::difference_type difference_type;
+		typedef typename allocator_type::size_type size_type;
 
 		explicit vector(const allocator_type& alloc = allocator_type())
 			: _alloc(alloc), _data(NULL), _size(0), _capacity(0)
@@ -70,8 +70,8 @@ namespace ft
 
 		iterator begin() { return _data; }
 		const_iterator begin() const { return _data; }
-		iterator end() { return _data + _size; }
-		const_iterator end() const { return _data + _size; }
+		iterator end() { return _iterator_at(_size); }
+		const_iterator end() const { return _iterator_at(_size); }
 
 		reverse_iterator rbegin() { return reverse_iterator(end()); }
 		const_reverse_iterator rbegin() const
@@ -176,16 +176,16 @@ namespace ft
 
 		iterator insert(iterator pos, const value_type& value)
 		{
-			size_type index = static_cast<size_type>(pos - begin());
+			size_type index = _index_of(pos);
 			insert(pos, 1, value);
-			return begin() + index;
+			return _iterator_at(index);
 		}
 
 		void insert(iterator pos, size_type count, const value_type& value)
 		{
-			size_type index = static_cast<size_type>(pos - begin());
 			if (count == 0)
 				return;
+			size_type index = _index_of(pos);
 			if (count > max_size() - _size)
 				throw std::length_error("ft::vector::insert");
 			if (_size + count > _capacity)
@@ -204,7 +204,7 @@ namespace ft
 		void insert(iterator pos, InputIt first, InputIt last,
 			typename ft::enable_if<!ft::is_integral<InputIt>::value>::type* = 0)
 		{
-			size_type index = static_cast<size_type>(pos - begin());
+			size_type index = _index_of(pos);
 			vector tmp(first, last, _alloc);
 			if (tmp.empty())
 				return;
@@ -226,14 +226,15 @@ namespace ft
 
 		iterator erase(iterator first, iterator last)
 		{
-			size_type index = static_cast<size_type>(first - begin());
-			size_type count = static_cast<size_type>(last - first);
+			size_type index = _index_of(first);
+			size_type count = first == last
+				? 0 : static_cast<size_type>(last - first);
 			for (size_type i = index; i + count < _size; ++i)
 				_data[i] = _data[i + count];
 			for (size_type i = 0; i < count; ++i)
 				_alloc.destroy(_data + _size - 1 - i);
 			_size -= count;
-			return begin() + index;
+			return _iterator_at(index);
 		}
 
 		void clear()
@@ -275,6 +276,21 @@ namespace ft
 			return next;
 		}
 
+		iterator _iterator_at(size_type index)
+		{
+			return _data ? _data + index : _data;
+		}
+
+		const_iterator _iterator_at(size_type index) const
+		{
+			return _data ? _data + index : _data;
+		}
+
+		size_type _index_of(const_iterator pos) const
+		{
+			return _data ? static_cast<size_type>(pos - _data) : 0;
+		}
+
 		void _reallocate(size_type new_cap)
 		{
 			pointer new_data = _alloc.allocate(new_cap);


## `test(vector): 빈 저장소와 allocator 상태 검증`

diff --git a/tests/test_vector_exceptions.cpp b/tests/test_vector_exceptions.cpp
index ef31479..a1c6b97 100644
--- a/tests/test_vector_exceptions.cpp
+++ b/tests/test_vector_exceptions.cpp
@@ -105,6 +105,9 @@ namespace
 		typedef ft::vector<int, int_allocator> int_vector;
 		{
 			int_vector values;
+			require(values.begin() == values.end(), "empty begin equals end");
+			values.insert(values.end(), 0, 4);
+			values.erase(values.begin(), values.end());
 			values.reserve(3);
 			for (int i = 0; i < 4; ++i)
 				values.push_back(i);
