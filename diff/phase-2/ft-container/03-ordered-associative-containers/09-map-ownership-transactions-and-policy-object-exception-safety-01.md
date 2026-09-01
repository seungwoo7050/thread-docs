# 맵 소유권 트랜잭션과 정책 객체 예외 안전성

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


## `fix(map): 값 allocator 상태로 노드 allocator 구성`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 8f7b154..710d7b7 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -219,7 +219,7 @@ namespace ft
 
 		explicit map(const key_compare& comp = key_compare(),
 			const allocator_type& alloc = allocator_type())
-			: _alloc(alloc), _node_alloc(node_allocator()), _root(NULL),
+			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _root(NULL),
 			  _size(0), _comp(comp)
 		{
 		}
@@ -227,14 +227,14 @@ namespace ft
 		template <class InputIt>
 		map(InputIt first, InputIt last, const key_compare& comp = key_compare(),
 			const allocator_type& alloc = allocator_type())
-			: _alloc(alloc), _node_alloc(node_allocator()), _root(NULL),
+			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _root(NULL),
 			  _size(0), _comp(comp)
 		{
 			insert(first, last);
 		}
 
 		map(const map& other)
-			: _alloc(other._alloc), _node_alloc(node_allocator()), _root(NULL),
+			: _alloc(other._alloc), _node_alloc(node_allocator(other._alloc)), _root(NULL),
 			  _size(0), _comp(other._comp)
 		{
 			insert(other.begin(), other.end());


## `test(map): 반복 삭제·복사·대입·교환 stress 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 8e0a203..03ecabf 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -294,6 +294,65 @@ namespace
 			insert_key(ftdesc, stddesc, i);
 		compare_map(ftdesc, stddesc, "map descending insert");
 		check_map_queries(ftdesc, stddesc, "map descending queries");
+
+		ft::map<int, std::string> ftcopy(ftdesc);
+		std::map<int, std::string> stdcopy(stddesc);
+		compare_map(ftcopy, stdcopy, "map copy constructor stress");
+
+		ft::map<int, std::string> ftassigned;
+		std::map<int, std::string> stdassigned;
+		ftassigned = ftasc;
+		stdassigned = stdasc;
+		compare_map(ftassigned, stdassigned, "map assignment stress");
+
+		ftassigned.swap(ftcopy);
+		stdassigned.swap(stdcopy);
+		compare_map(ftassigned, stdassigned, "map member swap lhs");
+		compare_map(ftcopy, stdcopy, "map member swap rhs");
+	}
+
+	void test_map_stress_erase()
+	{
+		ft::map<int, std::string> ftm;
+		std::map<int, std::string> stdm;
+		int keys[] = {
+			42, 7, 88, 13, 64, 2, 91, 55, 31, 76, 19, 4, 68, 27, 83, 10,
+			47, 99, 1, 35, 72, 58, 24, 90, 6, 40, 80, 15, 62, 30, 95, 50
+		};
+		for (std::size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); ++i)
+			insert_key(ftm, stdm, keys[i]);
+		compare_map(ftm, stdm, "map mixed insert");
+		check_map_queries(ftm, stdm, "map mixed queries");
+
+		for (std::size_t i = 0; i < sizeof(keys) / sizeof(keys[0]); i += 3)
+		{
+			require(ftm.erase(keys[i]) == stdm.erase(keys[i]),
+				"map erase key count");
+			compare_map(ftm, stdm, "map repeated key erase");
+			check_map_queries(ftm, stdm, "map repeated key erase queries");
+		}
+
+		while (!stdm.empty())
+		{
+			ft::map<int, std::string>::iterator fit;
+			std::map<int, std::string>::iterator sit;
+			if (stdm.size() % 2 == 0)
+			{
+				fit = ftm.begin();
+				sit = stdm.begin();
+			}
+			else
+			{
+				fit = ftm.end();
+				sit = stdm.end();
+				--fit;
+				--sit;
+			}
+			ftm.erase(fit);
+			stdm.erase(sit);
+			compare_map(ftm, stdm, "map repeated iterator erase");
+		}
+		require(ftm.empty() && stdm.empty(), "map erase to empty");
 	}
 }
 
@@ -304,6 +363,7 @@ int main()
 	test_stack();
 	test_map();
 	test_map_stress_ordering();
+	test_map_stress_erase();
 	std::cout << "ft_containers checks passed" << std::endl;
 	return 0;
 }


## `fix(map): 삽입 위치를 노드 할당 전에 확정`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 0f0f986..d0f3120 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -298,19 +298,26 @@ namespace ft
 			}
 			node* parent = NULL;
 			node* cur = _root;
+			bool insert_left = false;
 			while (cur)
 			{
 				parent = cur;
 				if (_comp(value.first, cur->value.first))
+				{
+					insert_left = true;
 					cur = cur->left;
+				}
 				else if (_comp(cur->value.first, value.first))
+				{
+					insert_left = false;
 					cur = cur->right;
+				}
 				else
 					return ft::make_pair(iterator(cur, _root), false);
 			}
 			node* created = _create_node(value);
 			created->parent = parent;
-			if (_comp(value.first, parent->value.first))
+			if (insert_left)
 				parent->left = created;
 			else
 				parent->right = created;


## `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index d0f3120..6719a3f 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -231,14 +231,30 @@ namespace ft
 			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _root(NULL),
 			  _size(0), _comp(comp)
 		{
-			insert(first, last);
+			try
+			{
+				insert(first, last);
+			}
+			catch (...)
+			{
+				clear();
+				throw;
+			}
 		}
 
 		map(const map& other)
 			: _alloc(other._alloc), _node_alloc(node_allocator(other._alloc)), _root(NULL),
 			  _size(0), _comp(other._comp)
 		{
-			insert(other.begin(), other.end());
+			try
+			{
+				insert(other.begin(), other.end());
+			}
+			catch (...)
+			{
+				clear();
+				throw;
+			}
 		}
 
 		~map()
@@ -250,9 +266,8 @@ namespace ft
 		{
 			if (this != &other)
 			{
-				clear();
-				_comp = other._comp;
-				insert(other.begin(), other.end());
+				map tmp(other.begin(), other.end(), other._comp, _alloc);
+				_swap_tree_and_compare(tmp);
 			}
 			return *this;
 		}
@@ -439,6 +454,13 @@ namespace ft
 		size_type _size;
 		key_compare _comp;
 
+		void _swap_tree_and_compare(map& other)
+		{
+			std::swap(_root, other._root);
+			std::swap(_size, other._size);
+			std::swap(_comp, other._comp);
+		}
+
 		node* _create_node(const value_type& value)
 		{
 			node* n = _node_alloc.allocate(1);


## `test(map): 비교·할당 실패 시 노드 소유권 검증`

diff --git a/Makefile b/Makefile
index 1a8ce10..d9f67ec 100644
--- a/Makefile
+++ b/Makefile
@@ -3,7 +3,8 @@ CXXFLAGS := -Wall -Wextra -Werror -std=c++98
 CPPFLAGS := -Iinclude
 
 BUILD_DIR := build
-TEST_NAMES := test_containers test_vector_exceptions test_map_iterators
+TEST_NAMES := test_containers test_vector_exceptions test_map_exceptions \
+	test_map_iterators
 TEST_BINS := $(addprefix $(BUILD_DIR)/,$(TEST_NAMES))
 HEADERS := $(wildcard include/*.hpp)
 
diff --git a/tests/test_map_exceptions.cpp b/tests/test_map_exceptions.cpp
new file mode 100644
index 0000000..7dc0962
--- /dev/null
+++ b/tests/test_map_exceptions.cpp
@@ -0,0 +1,399 @@
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
+	class injected_failure : public std::exception
+	{
+	public:
+		const char* what() const throw()
+		{
+			return "injected map failure";
+		}
+	};
+
+	struct comparison_state
+	{
+		int calls;
+		int throw_on_call;
+
+		comparison_state() : calls(0), throw_on_call(-1)
+		{
+		}
+
+		void reset()
+		{
+			calls = 0;
+			throw_on_call = -1;
+		}
+	};
+
+	class throwing_less
+	{
+	public:
+		explicit throwing_less(comparison_state* state = NULL) : _state(state)
+		{
+		}
+
+		bool operator()(int lhs, int rhs) const
+		{
+			if (_state != NULL
+				&& _state->calls++ == _state->throw_on_call)
+				throw injected_failure();
+			return lhs < rhs;
+		}
+
+	private:
+		comparison_state* _state;
+	};
+
+	struct allocation_state
+	{
+		int outstanding_blocks;
+		int allocation_calls;
+		int throw_on_call;
+
+		allocation_state()
+			: outstanding_blocks(0), allocation_calls(0), throw_on_call(-1)
+		{
+		}
+
+		void reset_failure()
+		{
+			allocation_calls = 0;
+			throw_on_call = -1;
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
+		explicit tracking_allocator(allocation_state* state = NULL)
+			: _state(state)
+		{
+		}
+
+		template <class U>
+		tracking_allocator(const tracking_allocator<U>& other)
+			: _state(other.state())
+		{
+		}
+
+		pointer allocate(size_type count, const void* = 0)
+		{
+			if (_state != NULL
+				&& _state->allocation_calls++ == _state->throw_on_call)
+				throw std::bad_alloc();
+			pointer result = std::allocator<T>().allocate(count);
+			if (_state != NULL)
+				++_state->outstanding_blocks;
+			return result;
+		}
+
+		void deallocate(pointer data, size_type count)
+		{
+			std::allocator<T>().deallocate(data, count);
+			if (_state != NULL)
+				--_state->outstanding_blocks;
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
+		allocation_state* state() const
+		{
+			return _state;
+		}
+
+	private:
+		allocation_state* _state;
+	};
+
+	template <class T, class U>
+	bool operator==(const tracking_allocator<T>& lhs,
+		const tracking_allocator<U>& rhs)
+	{
+		return lhs.state() == rhs.state();
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
+	typedef ft::map<int, int, throwing_less, map_allocator> tested_map;
+
+	class generated_iterator
+	{
+	public:
+		generated_iterator(const int* keys, std::size_t index)
+			: _keys(keys), _index(index)
+		{
+		}
+
+		map_value operator*() const
+		{
+			return map_value(_keys[_index], _keys[_index] * 10);
+		}
+
+		generated_iterator& operator++()
+		{
+			++_index;
+			return *this;
+		}
+
+		bool operator==(const generated_iterator& other) const
+		{
+			return _keys == other._keys && _index == other._index;
+		}
+
+		bool operator!=(const generated_iterator& other) const
+		{
+			return !(*this == other);
+		}
+
+	private:
+		const int* _keys;
+		std::size_t _index;
+	};
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
+		for (std::size_t i = 0; i < count; ++i, ++it)
+		{
+			require(it != values.end(), label + " early end");
+			require(it->first == expected[i], label + " key");
+		}
+		require(it == values.end(), label + " final end");
+	}
+
+	void test_insert_does_not_compare_after_allocation()
+	{
+		for (int fail_at = 0; fail_at < 5; ++fail_at)
+		{
+			comparison_state comparisons;
+			allocation_state allocations;
+			{
+				tested_map values((throwing_less(&comparisons)),
+					map_allocator(&allocations));
+				values.insert(map_value(10, 100));
+				comparisons.calls = 0;
+				comparisons.throw_on_call = fail_at;
+				bool thrown = false;
+				try
+				{
+					values.insert(map_value(15, 150));
+				}
+				catch (const injected_failure&)
+				{
+					thrown = true;
+				}
+				comparisons.reset();
+				if (thrown)
+				{
+					const int expected[] = {10};
+					require_keys(values, expected, 1,
+						"failed insert preserves tree");
+				}
+				else
+				{
+					const int expected[] = {10, 15};
+					require_keys(values, expected, 2,
+						"successful insert");
+				}
+				require(allocations.outstanding_blocks
+						== static_cast<int>(values.size()),
+					"insert owns every allocated node");
+			}
+			require(allocations.outstanding_blocks == 0,
+				"insert releases all nodes");
+		}
+	}
+
+	void test_range_constructor_rollback()
+	{
+		const int keys[] = {1, 2, 3, 4, 5, 6};
+		for (int fail_at = 0; fail_at < 18; ++fail_at)
+		{
+			comparison_state comparisons;
+			allocation_state allocations;
+			comparisons.throw_on_call = fail_at;
+			try
+			{
+				tested_map values(generated_iterator(keys, 0),
+					generated_iterator(keys, 6), throwing_less(&comparisons),
+					map_allocator(&allocations));
+			}
+			catch (const injected_failure&)
+			{
+			}
+			require(allocations.outstanding_blocks == 0,
+				"range constructor rolls back nodes");
+		}
+
+		for (int fail_at = 0; fail_at < 7; ++fail_at)
+		{
+			comparison_state comparisons;
+			allocation_state allocations;
+			allocations.throw_on_call = fail_at;
+			try
+			{
+				tested_map values(generated_iterator(keys, 0),
+					generated_iterator(keys, 6), throwing_less(&comparisons),
+					map_allocator(&allocations));
+			}
+			catch (const std::bad_alloc&)
+			{
+			}
+			require(allocations.outstanding_blocks == 0,
+				"range constructor handles allocation failure");
+		}
+	}
+
+	void test_copy_constructor_rollback()
+	{
+		comparison_state comparisons;
+		allocation_state allocations;
+		{
+			tested_map source((throwing_less(&comparisons)),
+				map_allocator(&allocations));
+			for (int key = 1; key <= 6; ++key)
+				source.insert(map_value(key, key * 10));
+			const int source_blocks = allocations.outstanding_blocks;
+			comparisons.calls = 0;
+			comparisons.throw_on_call = 2;
+			bool thrown = false;
+			try
+			{
+				tested_map copy(source);
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			comparisons.reset();
+			require(thrown, "copy constructor injects a failure");
+			require(allocations.outstanding_blocks == source_blocks,
+				"copy constructor rolls back nodes");
+		}
+		require(allocations.outstanding_blocks == 0,
+			"copy constructor source releases nodes");
+	}
+
+	void test_assignment_preserves_original()
+	{
+		comparison_state source_comparisons;
+		comparison_state target_comparisons;
+		allocation_state allocations;
+		{
+			tested_map source((throwing_less(&source_comparisons)),
+				map_allocator(&allocations));
+			for (int key = 1; key <= 6; ++key)
+				source.insert(map_value(key, key * 10));
+
+			tested_map target((throwing_less(&target_comparisons)),
+				map_allocator(&allocations));
+			target.insert(map_value(40, 400));
+			target.insert(map_value(50, 500));
+			const int expected[] = {40, 50};
+			const int baseline_blocks = allocations.outstanding_blocks;
+
+			source_comparisons.calls = 0;
+			source_comparisons.throw_on_call = 3;
+			bool thrown = false;
+			try
+			{
+				target = source;
+			}
+			catch (const injected_failure&)
+			{
+				thrown = true;
+			}
+			source_comparisons.reset();
+			target_comparisons.reset();
+			require(thrown, "assignment injects a comparator failure");
+			require_keys(target, expected, 2,
+				"assignment preserves original keys");
+			require(allocations.outstanding_blocks == baseline_blocks,
+				"assignment rolls back temporary nodes");
+
+			allocations.reset_failure();
+			allocations.throw_on_call = 2;
+			thrown = false;
+			try
+			{
+				target = source;
+			}
+			catch (const std::bad_alloc&)
+			{
+				thrown = true;
+			}
+			allocations.reset_failure();
+			require(thrown, "assignment injects an allocation failure");
+			require_keys(target, expected, 2,
+				"allocation failure preserves original keys");
+			require(allocations.outstanding_blocks == baseline_blocks,
+				"allocation failure rolls back temporary nodes");
+		}
+		require(allocations.outstanding_blocks == 0,
+			"assignment releases all nodes");
+	}
+}
+
+int main()
+{
+	test_insert_does_not_compare_after_allocation();
+	test_range_constructor_rollback();
+	test_copy_constructor_rollback();
+	test_assignment_preserves_original();
+	std::cout << "map exception checks passed" << std::endl;
+	return 0;
+}


