# 비교자 기반 정렬 맵 인터페이스와 탐색 파이프라인

## `feat(map): 노드 소유권과 빈 tree 상태 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
new file mode 100644
index 0000000..be547a2
--- /dev/null
+++ b/include/ft_map.hpp
@@ -0,0 +1,105 @@
+#ifndef FT_MAP_HPP
+# define FT_MAP_HPP
+
+# include <algorithm>
+# include <cstddef>
+# include <functional>
+# include <memory>
+# include "ft_algorithm.hpp"
+# include "ft_iterator.hpp"
+# include "ft_pair.hpp"
+
+namespace ft
+{
+	template <class Key, class T, class Compare = std::less<Key>,
+		class Alloc = std::allocator<ft::pair<const Key, T> > >
+	class map
+	{
+	public:
+		typedef Key key_type;
+		typedef T mapped_type;
+		typedef ft::pair<const key_type, mapped_type> value_type;
+		typedef Compare key_compare;
+		typedef Alloc allocator_type;
+		typedef value_type& reference;
+		typedef const value_type& const_reference;
+		typedef typename allocator_type::pointer pointer;
+		typedef typename allocator_type::const_pointer const_pointer;
+		typedef std::ptrdiff_t difference_type;
+		typedef std::size_t size_type;
+
+	private:
+		struct node
+		{
+			value_type value;
+			node* parent;
+			node* left;
+			node* right;
+
+			explicit node(const value_type& v)
+				: value(v), parent(NULL), left(NULL), right(NULL)
+			{
+			}
+		};
+
+		typedef typename allocator_type::template rebind<node>::other node_allocator;
+
+	public:
+		explicit map(const key_compare& comp = key_compare(),
+			const allocator_type& alloc = allocator_type())
+			: _alloc(alloc), _node_alloc(node_allocator()), _root(NULL),
+			  _size(0), _comp(comp)
+		{
+		}
+
+		~map()
+		{
+			_clear(_root);
+		}
+
+		bool empty() const { return _size == 0; }
+		size_type size() const { return _size; }
+		size_type max_size() const { return _node_alloc.max_size(); }
+		key_compare key_comp() const { return _comp; }
+		allocator_type get_allocator() const { return _alloc; }
+
+	private:
+		allocator_type _alloc;
+		node_allocator _node_alloc;
+		node* _root;
+		size_type _size;
+		key_compare _comp;
+
+		node* _create_node(const value_type& value)
+		{
+			node* n = _node_alloc.allocate(1);
+			try
+			{
+				_node_alloc.construct(n, node(value));
+			}
+			catch (...)
+			{
+				_node_alloc.deallocate(n, 1);
+				throw;
+			}
+			return n;
+		}
+
+		void _destroy_node(node* n)
+		{
+			_node_alloc.destroy(n);
+			_node_alloc.deallocate(n, 1);
+		}
+
+		void _clear(node* n)
+		{
+			if (n == NULL)
+				return;
+			_clear(n->left);
+			_clear(n->right);
+			_destroy_node(n);
+		}
+	};
+}
+
+#endif


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


## `feat(map): 삽입과 첨자 및 복사 동작 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 9e9a3e4..ac95c56 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -184,11 +184,40 @@ namespace ft
 		{
 		}
 
+		template <class InputIt>
+		map(InputIt first, InputIt last, const key_compare& comp = key_compare(),
+			const allocator_type& alloc = allocator_type())
+			: _alloc(alloc), _node_alloc(node_allocator()), _root(NULL),
+			  _size(0), _comp(comp)
+		{
+			insert(first, last);
+		}
+
+		map(const map& other)
+			: _alloc(other._alloc), _node_alloc(node_allocator()), _root(NULL),
+			  _size(0), _comp(other._comp)
+		{
+			insert(other.begin(), other.end());
+		}
+
 		~map()
 		{
 			_clear(_root);
 		}
 
+		map& operator=(const map& other)
+		{
+			if (this != &other)
+			{
+				_clear(_root);
+				_root = NULL;
+				_size = 0;
+				_comp = other._comp;
+				insert(other.begin(), other.end());
+			}
+			return *this;
+		}
+
 		iterator begin() { return iterator(_minimum(_root), _root); }
 		const_iterator begin() const
 		{
@@ -213,6 +242,55 @@ namespace ft
 		bool empty() const { return _size == 0; }
 		size_type size() const { return _size; }
 		size_type max_size() const { return _node_alloc.max_size(); }
+
+		mapped_type& operator[](const key_type& key)
+		{
+			return insert(value_type(key, mapped_type())).first->second;
+		}
+
+		ft::pair<iterator, bool> insert(const value_type& value)
+		{
+			if (_root == NULL)
+			{
+				_root = _create_node(value);
+				++_size;
+				return ft::make_pair(iterator(_root, _root), true);
+			}
+			node* parent = NULL;
+			node* cur = _root;
+			while (cur)
+			{
+				parent = cur;
+				if (_comp(value.first, cur->value.first))
+					cur = cur->left;
+				else if (_comp(cur->value.first, value.first))
+					cur = cur->right;
+				else
+					return ft::make_pair(iterator(cur, _root), false);
+			}
+			node* created = _create_node(value);
+			created->parent = parent;
+			if (_comp(value.first, parent->value.first))
+				parent->left = created;
+			else
+				parent->right = created;
+			++_size;
+			return ft::make_pair(iterator(created, _root), true);
+		}
+
+		iterator insert(iterator hint, const value_type& value)
+		{
+			(void)hint;
+			return insert(value).first;
+		}
+
+		template <class InputIt>
+		void insert(InputIt first, InputIt last)
+		{
+			for (; first != last; ++first)
+				insert(*first);
+		}
+
 		key_compare key_comp() const { return _comp; }
 		allocator_type get_allocator() const { return _alloc; }
 


## `feat(map): 검색과 경계 query 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index ac95c56..457a0a8 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -294,6 +294,52 @@ namespace ft
 		key_compare key_comp() const { return _comp; }
 		allocator_type get_allocator() const { return _alloc; }
 
+		iterator find(const key_type& key)
+		{
+			return iterator(_find_node(key), _root);
+		}
+
+		const_iterator find(const key_type& key) const
+		{
+			return const_iterator(_find_node(key), _root);
+		}
+
+		size_type count(const key_type& key) const
+		{
+			return _find_node(key) ? 1 : 0;
+		}
+
+		iterator lower_bound(const key_type& key)
+		{
+			return iterator(_lower_bound_node(key), _root);
+		}
+
+		const_iterator lower_bound(const key_type& key) const
+		{
+			return const_iterator(_lower_bound_node(key), _root);
+		}
+
+		iterator upper_bound(const key_type& key)
+		{
+			return iterator(_upper_bound_node(key), _root);
+		}
+
+		const_iterator upper_bound(const key_type& key) const
+		{
+			return const_iterator(_upper_bound_node(key), _root);
+		}
+
+		ft::pair<iterator, iterator> equal_range(const key_type& key)
+		{
+			return ft::make_pair(lower_bound(key), upper_bound(key));
+		}
+
+		ft::pair<const_iterator, const_iterator> equal_range(
+			const key_type& key) const
+		{
+			return ft::make_pair(lower_bound(key), upper_bound(key));
+		}
+
 	private:
 		allocator_type _alloc;
 		node_allocator _node_alloc;
@@ -370,6 +416,55 @@ namespace ft
 			return parent;
 		}
 
+		node* _find_node(const key_type& key) const
+		{
+			node* cur = _root;
+			while (cur)
+			{
+				if (_comp(key, cur->value.first))
+					cur = cur->left;
+				else if (_comp(cur->value.first, key))
+					cur = cur->right;
+				else
+					return cur;
+			}
+			return NULL;
+		}
+
+		node* _lower_bound_node(const key_type& key) const
+		{
+			node* cur = _root;
+			node* result = NULL;
+			while (cur)
+			{
+				if (!_comp(cur->value.first, key))
+				{
+					result = cur;
+					cur = cur->left;
+				}
+				else
+					cur = cur->right;
+			}
+			return result;
+		}
+
+		node* _upper_bound_node(const key_type& key) const
+		{
+			node* cur = _root;
+			node* result = NULL;
+			while (cur)
+			{
+				if (_comp(key, cur->value.first))
+				{
+					result = cur;
+					cur = cur->left;
+				}
+				else
+					cur = cur->right;
+			}
+			return result;
+		}
+
 		void _clear(node* n)
 		{
 			if (n == NULL)


## `feat(map): 삭제와 clear 및 swap 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 457a0a8..86aa7d1 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -202,16 +202,14 @@ namespace ft
 
 		~map()
 		{
-			_clear(_root);
+			clear();
 		}
 
 		map& operator=(const map& other)
 		{
 			if (this != &other)
 			{
-				_clear(_root);
-				_root = NULL;
-				_size = 0;
+				clear();
 				_comp = other._comp;
 				insert(other.begin(), other.end());
 			}
@@ -291,6 +289,49 @@ namespace ft
 				insert(*first);
 		}
 
+		void erase(iterator pos)
+		{
+			if (pos == end())
+				return;
+			_erase_node(pos._node);
+		}
+
+		size_type erase(const key_type& key)
+		{
+			iterator it = find(key);
+			if (it == end())
+				return 0;
+			erase(it);
+			return 1;
+		}
+
+		void erase(iterator first, iterator last)
+		{
+			while (first != last)
+			{
+				iterator next = first;
+				++next;
+				erase(first);
+				first = next;
+			}
+		}
+
+		void clear()
+		{
+			_clear(_root);
+			_root = NULL;
+			_size = 0;
+		}
+
+		void swap(map& other)
+		{
+			std::swap(_alloc, other._alloc);
+			std::swap(_node_alloc, other._node_alloc);
+			std::swap(_root, other._root);
+			std::swap(_size, other._size);
+			std::swap(_comp, other._comp);
+		}
+
 		key_compare key_comp() const { return _comp; }
 		allocator_type get_allocator() const { return _alloc; }
 
@@ -465,6 +506,41 @@ namespace ft
 			return result;
 		}
 
+		void _transplant(node* old_node, node* new_node)
+		{
+			if (old_node->parent == NULL)
+				_root = new_node;
+			else if (old_node == old_node->parent->left)
+				old_node->parent->left = new_node;
+			else
+				old_node->parent->right = new_node;
+			if (new_node)
+				new_node->parent = old_node->parent;
+		}
+
+		void _erase_node(node* target)
+		{
+			if (target->left == NULL)
+				_transplant(target, target->right);
+			else if (target->right == NULL)
+				_transplant(target, target->left);
+			else
+			{
+				node* successor = _minimum(target->right);
+				if (successor->parent != target)
+				{
+					_transplant(successor, successor->right);
+					successor->right = target->right;
+					successor->right->parent = successor;
+				}
+				_transplant(target, successor);
+				successor->left = target->left;
+				successor->left->parent = successor;
+			}
+			_destroy_node(target);
+			--_size;
+		}
+
 		void _clear(node* n)
 		{
 			if (n == NULL)


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


## `test(map): 삽입·검색·삭제 결과를 표준 map과 비교`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 318e81e..dfb2319 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -1,5 +1,7 @@
 #include <cstdlib>
 #include <iostream>
+#include <map>
+#include <sstream>
 #include <stack>
 #include <stdexcept>
 #include <string>
@@ -119,6 +121,65 @@ namespace
 		require((fta == ftb) == (stda == stdb), "stack equality compare");
 		require((fta < ftb) == (stda < stdb), "stack less compare");
 	}
+
+	template <class FtMap, class StdMap>
+	void compare_map(const FtMap& ftm, const StdMap& stdm,
+		const std::string& label)
+	{
+		require(ftm.size() == stdm.size(), label + " size");
+		typename FtMap::const_iterator fit = ftm.begin();
+		typename StdMap::const_iterator sit = stdm.begin();
+		for (; fit != ftm.end() && sit != stdm.end(); ++fit, ++sit)
+		{
+			require(fit->first == sit->first, label + " key order");
+			require(fit->second == sit->second, label + " mapped value");
+		}
+		require(fit == ftm.end() && sit == stdm.end(), label + " end");
+	}
+
+	void test_map()
+	{
+		ft::map<int, std::string> ftm;
+		std::map<int, std::string> stdm;
+		int keys[] = {8, 3, 10, 1, 6, 14, 4, 7, 13, 6, 8};
+		for (std::size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); ++i)
+		{
+			std::ostringstream oss;
+			oss << "v" << keys[i];
+			ft::pair<ft::map<int, std::string>::iterator, bool> fr =
+				ftm.insert(ft::make_pair(keys[i], oss.str()));
+			std::pair<std::map<int, std::string>::iterator, bool> sr =
+				stdm.insert(std::make_pair(keys[i], oss.str()));
+			require(fr.second == sr.second, "map duplicate insert flag");
+		}
+		compare_map(ftm, stdm, "map insert order");
+
+		ftm[5] = "five";
+		stdm[5] = "five";
+		ftm[6] = "six";
+		stdm[6] = "six";
+		compare_map(ftm, stdm, "map operator[]");
+
+		require(ftm.find(7)->second == stdm.find(7)->second, "map find");
+		require(ftm.count(111) == stdm.count(111), "map count missing");
+		require(ftm.lower_bound(6)->first == stdm.lower_bound(6)->first,
+			"map lower_bound");
+		require(ftm.upper_bound(6)->first == stdm.upper_bound(6)->first,
+			"map upper_bound");
+		require(ftm.equal_range(6).first->first == stdm.equal_range(6).first->first,
+			"map equal_range first");
+
+		ftm.erase(3);
+		stdm.erase(3);
+		ftm.erase(ftm.find(10));
+		stdm.erase(stdm.find(10));
+		compare_map(ftm, stdm, "map erase");
+
+		ft::map<int, std::string> ftcopy(ftm.begin(), ftm.end());
+		std::map<int, std::string> stdcopy(stdm.begin(), stdm.end());
+		compare_map(ftcopy, stdcopy, "map range constructor");
+		require(ftcopy == ftm, "map equality");
+	}
 }
 
 int main()
@@ -126,6 +187,7 @@ int main()
 	test_utilities();
 	test_vector();
 	test_stack();
+	test_map();
 	std::cout << "ft_containers checks passed" << std::endl;
 	return 0;
 }


