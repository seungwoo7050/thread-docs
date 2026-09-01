# 헤더 센티널, `end` 표현과 맵 반복자 안정성

## `feat(map): 가변 반복자와 tree 순회 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index be547a2..5a3d455 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -42,9 +42,70 @@ namespace ft
 			}
 		};
 
-		typedef typename allocator_type::template rebind<node>::other node_allocator;
+	typedef typename allocator_type::template rebind<node>::other node_allocator;
 
 	public:
+		class iterator
+			: public ft::iterator<std::bidirectional_iterator_tag, value_type>
+		{
+			friend class map;
+
+		public:
+			iterator() : _node(NULL), _root(NULL)
+			{
+			}
+
+			reference operator*() const { return _node->value; }
+			pointer operator->() const { return &_node->value; }
+
+			iterator& operator++()
+			{
+				_node = _next(_node);
+				return *this;
+			}
+
+			iterator operator++(int)
+			{
+				iterator tmp(*this);
+				++(*this);
+				return tmp;
+			}
+
+			iterator& operator--()
+			{
+				if (_node == NULL)
+					_node = _maximum(_root);
+				else
+					_node = _previous(_node);
+				return *this;
+			}
+
+			iterator operator--(int)
+			{
+				iterator tmp(*this);
+				--(*this);
+				return tmp;
+			}
+
+			bool operator==(const iterator& other) const
+			{
+				return _node == other._node;
+			}
+
+			bool operator!=(const iterator& other) const
+			{
+				return !(*this == other);
+			}
+
+		private:
+			node* _node;
+			node* _root;
+
+			iterator(node* n, node* r) : _node(n), _root(r)
+			{
+			}
+		};
+
 		explicit map(const key_compare& comp = key_compare(),
 			const allocator_type& alloc = allocator_type())
 			: _alloc(alloc), _node_alloc(node_allocator()), _root(NULL),
@@ -57,6 +118,9 @@ namespace ft
 			_clear(_root);
 		}
 
+		iterator begin() { return iterator(_minimum(_root), _root); }
+		iterator end() { return iterator(NULL, _root); }
+
 		bool empty() const { return _size == 0; }
 		size_type size() const { return _size; }
 		size_type max_size() const { return _node_alloc.max_size(); }
@@ -91,6 +155,54 @@ namespace ft
 			_node_alloc.deallocate(n, 1);
 		}
 
+		static node* _minimum(node* n)
+		{
+			if (n == NULL)
+				return NULL;
+			while (n->left)
+				n = n->left;
+			return n;
+		}
+
+		static node* _maximum(node* n)
+		{
+			if (n == NULL)
+				return NULL;
+			while (n->right)
+				n = n->right;
+			return n;
+		}
+
+		static node* _next(node* n)
+		{
+			if (n == NULL)
+				return NULL;
+			if (n->right)
+				return _minimum(n->right);
+			node* parent = n->parent;
+			while (parent && n == parent->right)
+			{
+				n = parent;
+				parent = parent->parent;
+			}
+			return parent;
+		}
+
+		static node* _previous(node* n)
+		{
+			if (n == NULL)
+				return NULL;
+			if (n->left)
+				return _maximum(n->left);
+			node* parent = n->parent;
+			while (parent && n == parent->left)
+			{
+				n = parent;
+				parent = parent->parent;
+			}
+			return parent;
+		}
+
 		void _clear(node* n)
 		{
 			if (n == NULL)


## `feat(map): 상수와 역방향 반복자 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 5a3d455..9e9a3e4 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -49,6 +49,7 @@ namespace ft
 			: public ft::iterator<std::bidirectional_iterator_tag, value_type>
 		{
 			friend class map;
+			friend class const_iterator;
 
 		public:
 			iterator() : _node(NULL), _root(NULL)
@@ -106,6 +107,76 @@ namespace ft
 			}
 		};
 
+		class const_iterator
+			: public ft::iterator<std::bidirectional_iterator_tag,
+				const value_type>
+		{
+			friend class map;
+
+		public:
+			const_iterator() : _node(NULL), _root(NULL)
+			{
+			}
+
+			const_iterator(const iterator& other)
+				: _node(other._node), _root(other._root)
+			{
+			}
+
+			const_reference operator*() const { return _node->value; }
+			const_pointer operator->() const { return &_node->value; }
+
+			const_iterator& operator++()
+			{
+				_node = _next(_node);
+				return *this;
+			}
+
+			const_iterator operator++(int)
+			{
+				const_iterator tmp(*this);
+				++(*this);
+				return tmp;
+			}
+
+			const_iterator& operator--()
+			{
+				if (_node == NULL)
+					_node = _maximum(_root);
+				else
+					_node = _previous(_node);
+				return *this;
+			}
+
+			const_iterator operator--(int)
+			{
+				const_iterator tmp(*this);
+				--(*this);
+				return tmp;
+			}
+
+			bool operator==(const const_iterator& other) const
+			{
+				return _node == other._node;
+			}
+
+			bool operator!=(const const_iterator& other) const
+			{
+				return !(*this == other);
+			}
+
+		private:
+			node* _node;
+			node* _root;
+
+			const_iterator(node* n, node* r) : _node(n), _root(r)
+			{
+			}
+		};
+
+		typedef ft::reverse_iterator<iterator> reverse_iterator;
+		typedef ft::reverse_iterator<const_iterator> const_reverse_iterator;
+
 		explicit map(const key_compare& comp = key_compare(),
 			const allocator_type& alloc = allocator_type())
 			: _alloc(alloc), _node_alloc(node_allocator()), _root(NULL),
@@ -119,7 +190,25 @@ namespace ft
 		}
 
 		iterator begin() { return iterator(_minimum(_root), _root); }
+		const_iterator begin() const
+		{
+			return const_iterator(_minimum(_root), _root);
+		}
+
 		iterator end() { return iterator(NULL, _root); }
+		const_iterator end() const { return const_iterator(NULL, _root); }
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
 
 		bool empty() const { return _size == 0; }
 		size_type size() const { return _size; }


## `feat(map): 가변·상수 반복자 상호 비교 지원`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index de89781..8f7b154 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -178,11 +178,33 @@ namespace ft
 				return _node == other._node;
 			}
 
+			bool operator==(const iterator& other) const
+			{
+				return _node == other._node;
+			}
+
 			bool operator!=(const const_iterator& other) const
 			{
 				return !(*this == other);
 			}
 
+			bool operator!=(const iterator& other) const
+			{
+				return !(*this == other);
+			}
+
+			friend bool operator==(const iterator& lhs,
+				const const_iterator& rhs)
+			{
+				return const_iterator(lhs) == rhs;
+			}
+
+			friend bool operator!=(const iterator& lhs,
+				const const_iterator& rhs)
+			{
+				return !(lhs == rhs);
+			}
+
 		private:
 			node* _node;
 			node* _root;


## `test(map): 가변·상수 반복자 비교 검증`

diff --git a/Makefile b/Makefile
index 9cff901..1a8ce10 100644
--- a/Makefile
+++ b/Makefile
@@ -3,7 +3,7 @@ CXXFLAGS := -Wall -Wextra -Werror -std=c++98
 CPPFLAGS := -Iinclude
 
 BUILD_DIR := build
-TEST_NAMES := test_containers test_vector_exceptions
+TEST_NAMES := test_containers test_vector_exceptions test_map_iterators
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
 HEADERS := $(wildcard include/*.hpp)
 
diff --git a/tests/test_map_iterators.cpp b/tests/test_map_iterators.cpp
new file mode 100644
index 0000000..70a31f0
--- /dev/null
+++ b/tests/test_map_iterators.cpp
@@ -0,0 +1,40 @@
+#include <cstdlib>
+#include <iostream>
+#include <string>
+
+#include "ft_map.hpp"
+
+namespace
+{
+	void require(bool condition, const std::string& message)
+	{
+		if (!condition)
+		{
+			std::cerr << "FAIL: " << message << std::endl;
+			std::exit(1);
+		}
+	}
+
+	void test_mixed_iterator_comparisons()
+	{
+		ft::map<int, int> values;
+		values.insert(ft::make_pair(4, 40));
+		ft::map<int, int>::iterator mutable_it = values.begin();
+		ft::map<int, int>::const_iterator const_it = mutable_it;
+		require(mutable_it == const_it,
+			"iterator compares equal to const_iterator");
+		require(const_it == mutable_it,
+			"const_iterator compares equal to iterator");
+		require(!(mutable_it != const_it),
+			"iterator mixed inequality is symmetric");
+		require(!(const_it != mutable_it),
+			"const_iterator mixed inequality is symmetric");
+	}
+}
+
+int main()
+{
+	test_mixed_iterator_comparisons();
+	std::cout << "map iterator checks passed" << std::endl;
+	return 0;
+}


## `test(map): 역방향 순회와 경계 query 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 16451a6..48d1a3b 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -184,6 +184,18 @@ namespace
 			"map upper_bound");
 		require(ftm.equal_range(6).first->first == stdm.equal_range(6).first->first,
 			"map equal_range first");
+		require(ftm.lower_bound(2)->first == stdm.lower_bound(2)->first,
+			"map lower_bound gap");
+		require(ftm.upper_bound(13)->first == stdm.upper_bound(13)->first,
+			"map upper_bound near end");
+
+		ft::map<int, std::string>::reverse_iterator fmrit = ftm.rbegin();
+		std::map<int, std::string>::reverse_iterator smrit = stdm.rbegin();
+		for (; fmrit != ftm.rend() && smrit != stdm.rend(); ++fmrit, ++smrit)
+		{
+			require(fmrit->first == smrit->first, "map reverse key");
+			require(fmrit->second == smrit->second, "map reverse value");
+		}
 
 		ftm.erase(3);
 		stdm.erase(3);


## `test(map): 상수 begin과 reverse begin 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 48d1a3b..4ae588b 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -207,6 +207,13 @@ namespace
 		std::map<int, std::string> stdcopy(stdm.begin(), stdm.end());
 		compare_map(ftcopy, stdcopy, "map range constructor");
 		require(ftcopy == ftm, "map equality");
+
+		const ft::map<int, std::string>& ftconst = ftcopy;
+		const std::map<int, std::string>& stdconst = stdcopy;
+		require(ftconst.begin()->first == stdconst.begin()->first,
+			"map const begin");
+		require(ftconst.rbegin()->first == stdconst.rbegin()->first,
+			"map const rbegin");
 	}
 }
 


