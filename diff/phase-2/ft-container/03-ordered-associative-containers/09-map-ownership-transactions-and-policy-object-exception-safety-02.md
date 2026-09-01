## `fix(map): 값 없는 header로 끝 반복자 상태 안정화`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 6719a3f..87c690b 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -29,17 +29,29 @@ namespace ft
 		typedef std::size_t size_type;
 
 	private:
-		struct node
+		struct node_base
 		{
-			value_type value;
-			node* parent;
-			node* left;
-			node* right;
+			node_base* parent;
+			node_base* left;
+			node_base* right;
 			bool red;
+			bool is_header;
+
+			explicit node_base(bool header = false)
+				: parent(NULL), left(header ? this : NULL),
+				  right(header ? this : NULL), red(false), is_header(header)
+			{
+			}
+		};
+
+		struct node : node_base
+		{
+			value_type value;
 
 			explicit node(const value_type& v)
-				: value(v), parent(NULL), left(NULL), right(NULL), red(true)
+				: node_base(false), value(v)
 			{
+				this->red = true;
 			}
 		};
 
@@ -71,12 +83,19 @@ namespace ft
 			friend class const_iterator;
 
 		public:
-			iterator() : _node(NULL), _root(NULL)
+			iterator() : _node(NULL)
 			{
 			}
 
-			reference operator*() const { return _node->value; }
-			pointer operator->() const { return &_node->value; }
+			reference operator*() const
+			{
+				return static_cast<node*>(_node)->value;
+			}
+
+			pointer operator->() const
+			{
+				return &static_cast<node*>(_node)->value;
+			}
 
 			iterator& operator++()
 			{
@@ -93,10 +112,7 @@ namespace ft
 
 			iterator& operator--()
 			{
-				if (_node == NULL)
-					_node = _maximum(_root);
-				else
-					_node = _previous(_node);
+				_node = _previous(_node);
 				return *this;
 			}
 
@@ -107,21 +123,22 @@ namespace ft
 				return tmp;
 			}
 
-			bool operator==(const iterator& other) const
+			template <class OtherIterator>
+			bool operator==(const OtherIterator& other) const
 			{
 				return _node == other._node;
 			}
 
-			bool operator!=(const iterator& other) const
+			template <class OtherIterator>
+			bool operator!=(const OtherIterator& other) const
 			{
 				return !(*this == other);
 			}
 
 		private:
-			node* _node;
-			node* _root;
+			node_base* _node;
 
-			iterator(node* n, node* r) : _node(n), _root(r)
+			explicit iterator(node_base* n) : _node(n)
 			{
 			}
 		};
@@ -131,19 +148,27 @@ namespace ft
 				const value_type>
 		{
 			friend class map;
+			friend class iterator;
 
 		public:
-			const_iterator() : _node(NULL), _root(NULL)
+			const_iterator() : _node(NULL)
 			{
 			}
 
 			const_iterator(const iterator& other)
-				: _node(other._node), _root(other._root)
+				: _node(other._node)
 			{
 			}
 
-			const_reference operator*() const { return _node->value; }
-			const_pointer operator->() const { return &_node->value; }
+			const_reference operator*() const
+			{
+				return static_cast<const node*>(_node)->value;
+			}
+
+			const_pointer operator->() const
+			{
+				return &static_cast<const node*>(_node)->value;
+			}
 
 			const_iterator& operator++()
 			{
@@ -160,10 +185,7 @@ namespace ft
 
 			const_iterator& operator--()
 			{
-				if (_node == NULL)
-					_node = _maximum(_root);
-				else
-					_node = _previous(_node);
+				_node = _previous(_node);
 				return *this;
 			}
 
@@ -174,43 +196,22 @@ namespace ft
 				return tmp;
 			}
 
-			bool operator==(const const_iterator& other) const
+			template <class OtherIterator>
+			bool operator==(const OtherIterator& other) const
 			{
 				return _node == other._node;
 			}
 
-			bool operator==(const iterator& other) const
-			{
-				return _node == other._node;
-			}
-
-			bool operator!=(const const_iterator& other) const
-			{
-				return !(*this == other);
-			}
-
-			bool operator!=(const iterator& other) const
+			template <class OtherIterator>
+			bool operator!=(const OtherIterator& other) const
 			{
 				return !(*this == other);
 			}
 
-			friend bool operator==(const iterator& lhs,
-				const const_iterator& rhs)
-			{
-				return const_iterator(lhs) == rhs;
-			}
-
-			friend bool operator!=(const iterator& lhs,
-				const const_iterator& rhs)
-			{
-				return !(lhs == rhs);
-			}
-
 		private:
-			node* _node;
-			node* _root;
+			node_base* _node;
 
-			const_iterator(node* n, node* r) : _node(n), _root(r)
+			explicit const_iterator(node_base* n) : _node(n)
 			{
 			}
 		};
@@ -220,7 +221,7 @@ namespace ft
 
 		explicit map(const key_compare& comp = key_compare(),
 			const allocator_type& alloc = allocator_type())
-			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _root(NULL),
+			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _header(true),
 			  _size(0), _comp(comp)
 		{
 		}
@@ -228,7 +229,7 @@ namespace ft
 		template <class InputIt>
 		map(InputIt first, InputIt last, const key_compare& comp = key_compare(),
 			const allocator_type& alloc = allocator_type())
-			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _root(NULL),
+			: _alloc(alloc), _node_alloc(node_allocator(alloc)), _header(true),
 			  _size(0), _comp(comp)
 		{
 			try
@@ -243,8 +244,8 @@ namespace ft
 		}
 
 		map(const map& other)
-			: _alloc(other._alloc), _node_alloc(node_allocator(other._alloc)), _root(NULL),
-			  _size(0), _comp(other._comp)
+			: _alloc(other._alloc), _node_alloc(node_allocator(other._alloc)),
+			  _header(true), _size(0), _comp(other._comp)
 		{
 			try
 			{
@@ -272,14 +273,17 @@ namespace ft
 			return *this;
 		}
 
-		iterator begin() { return iterator(_minimum(_root), _root); }
+		iterator begin() { return iterator(_header.left); }
 		const_iterator begin() const
 		{
-			return const_iterator(_minimum(_root), _root);
+			return const_iterator(_header.left);
 		}
 
-		iterator end() { return iterator(NULL, _root); }
-		const_iterator end() const { return const_iterator(NULL, _root); }
+		iterator end() { return iterator(&_header); }
+		const_iterator end() const
+		{
+			return const_iterator(const_cast<node_base*>(&_header));
+		}
 
 		reverse_iterator rbegin() { return reverse_iterator(end()); }
 		const_reverse_iterator rbegin() const
@@ -304,33 +308,37 @@ namespace ft
 
 		ft::pair<iterator, bool> insert(const value_type& value)
 		{
-			if (_root == NULL)
+			if (_root() == NULL)
 			{
-				_root = _create_node(value);
-				_root->red = false;
+				node_base* created = _create_node(value);
+				created->red = false;
+				created->parent = &_header;
+				_header.parent = created;
+				_header.left = created;
+				_header.right = created;
 				++_size;
-				return ft::make_pair(iterator(_root, _root), true);
+				return ft::make_pair(iterator(created), true);
 			}
-			node* parent = NULL;
-			node* cur = _root;
+			node_base* parent = &_header;
+			node_base* cur = _root();
 			bool insert_left = false;
 			while (cur)
 			{
 				parent = cur;
-				if (_comp(value.first, cur->value.first))
+				if (_comp(value.first, _value(cur).first))
 				{
 					insert_left = true;
 					cur = cur->left;
 				}
-				else if (_comp(cur->value.first, value.first))
+				else if (_comp(_value(cur).first, value.first))
 				{
 					insert_left = false;
 					cur = cur->right;
 				}
 				else
-					return ft::make_pair(iterator(cur, _root), false);
+					return ft::make_pair(iterator(cur), false);
 			}
-			node* created = _create_node(value);
+			node_base* created = _create_node(value);
 			created->parent = parent;
 			if (insert_left)
 				parent->left = created;
@@ -338,7 +346,8 @@ namespace ft
 				parent->right = created;
 			_insert_fixup(created);
 			++_size;
-			return ft::make_pair(iterator(created, _root), true);
+			_refresh_header();
+			return ft::make_pair(iterator(created), true);
 		}
 
 		iterator insert(iterator hint, const value_type& value)
@@ -383,18 +392,20 @@ namespace ft
 
 		void clear()
 		{
-			_clear(_root);
-			_root = NULL;
+			_clear(_root());
 			_size = 0;
+			_reset_header();
 		}
 
 		void swap(map& other)
 		{
 			std::swap(_alloc, other._alloc);
 			std::swap(_node_alloc, other._node_alloc);
-			std::swap(_root, other._root);
+			std::swap(_header.parent, other._header.parent);
 			std::swap(_size, other._size);
 			std::swap(_comp, other._comp);
+			_refresh_header();
+			other._refresh_header();
 		}
 
 		key_compare key_comp() const { return _comp; }
@@ -403,12 +414,15 @@ namespace ft
 
 		iterator find(const key_type& key)
 		{
-			return iterator(_find_node(key), _root);
+			node_base* found = _find_node(key);
+			return iterator(found ? found : &_header);
 		}
 
 		const_iterator find(const key_type& key) const
 		{
-			return const_iterator(_find_node(key), _root);
+			node_base* found = _find_node(key);
+			return const_iterator(found ? found
+				: const_cast<node_base*>(&_header));
 		}
 
 		size_type count(const key_type& key) const
@@ -418,22 +432,28 @@ namespace ft
 
 		iterator lower_bound(const key_type& key)
 		{
-			return iterator(_lower_bound_node(key), _root);
+			node_base* found = _lower_bound_node(key);
+			return iterator(found ? found : &_header);
 		}
 
 		const_iterator lower_bound(const key_type& key) const
 		{
-			return const_iterator(_lower_bound_node(key), _root);
+			node_base* found = _lower_bound_node(key);
+			return const_iterator(found ? found
+				: const_cast<node_base*>(&_header));
 		}
 
 		iterator upper_bound(const key_type& key)
 		{
-			return iterator(_upper_bound_node(key), _root);
+			node_base* found = _upper_bound_node(key);
+			return iterator(found ? found : &_header);
 		}
 
 		const_iterator upper_bound(const key_type& key) const
 		{
-			return const_iterator(_upper_bound_node(key), _root);
+			node_base* found = _upper_bound_node(key);
+			return const_iterator(found ? found
+				: const_cast<node_base*>(&_header));
 		}
 
 		ft::pair<iterator, iterator> equal_range(const key_type& key)
@@ -450,15 +470,54 @@ namespace ft
 	private:
 		allocator_type _alloc;
 		node_allocator _node_alloc;
-		node* _root;
+		node_base _header;
 		size_type _size;
 		key_compare _comp;
 
 		void _swap_tree_and_compare(map& other)
 		{
-			std::swap(_root, other._root);
+			std::swap(_header.parent, other._header.parent);
 			std::swap(_size, other._size);
 			std::swap(_comp, other._comp);
+			_refresh_header();
+			other._refresh_header();
+		}
+
+		node_base* _root() const
+		{
+			return _header.parent;
+		}
+
+		static value_type& _value(node_base* current)
+		{
+			return static_cast<node*>(current)->value;
+		}
+
+		static const value_type& _value(const node_base* current)
+		{
+			return static_cast<const node*>(current)->value;
+		}
+
+		void _reset_header()
+		{
+			_header.parent = NULL;
+			_header.left = &_header;
+			_header.right = &_header;
+			_header.red = false;
+			_header.is_header = true;
+		}
+
+		void _refresh_header()
+		{
+			node_base* root = _root();
+			if (root == NULL)
+			{
+				_reset_header();
+				return;
+			}
+			root->parent = &_header;
+			_header.left = _minimum(root);
+			_header.right = _maximum(root);
 		}
 
 		node* _create_node(const value_type& value)
@@ -476,13 +535,14 @@ namespace ft
 			return n;
 		}
 
-		void _destroy_node(node* n)
+		void _destroy_node(node_base* n)
 		{
-			_node_alloc.destroy(n);
-			_node_alloc.deallocate(n, 1);
+			node* concrete = static_cast<node*>(n);
+			_node_alloc.destroy(concrete);
+			_node_alloc.deallocate(concrete, 1);
 		}
 
-		static node* _minimum(node* n)
+		static node_base* _minimum(node_base* n)
 		{
 			if (n == NULL)
 				return NULL;
@@ -491,7 +551,7 @@ namespace ft
 			return n;
 		}
 
-		static node* _maximum(node* n)
+		static node_base* _maximum(node_base* n)
 		{
 			if (n == NULL)
 				return NULL;
@@ -500,14 +560,16 @@ namespace ft
 			return n;
 		}
 
-		static node* _next(node* n)
+		static node_base* _next(node_base* n)
 		{
 			if (n == NULL)
 				return NULL;
+			if (n->is_header)
+				return n;
 			if (n->right)
 				return _minimum(n->right);
-			node* parent = n->parent;
-			while (parent && n == parent->right)
+			node_base* parent = n->parent;
+			while (!parent->is_header && n == parent->right)
 			{
 				n = parent;
 				parent = parent->parent;
@@ -515,14 +577,16 @@ namespace ft
 			return parent;
 		}
 
-		static node* _previous(node* n)
+		static node_base* _previous(node_base* n)
 		{
 			if (n == NULL)
 				return NULL;
+			if (n->is_header)
+				return n->right;
 			if (n->left)
 				return _maximum(n->left);
-			node* parent = n->parent;
-			while (parent && n == parent->left)
+			node_base* parent = n->parent;
+			while (!parent->is_header && n == parent->left)
 			{
 				n = parent;
 				parent = parent->parent;
@@ -530,25 +594,25 @@ namespace ft
 			return parent;
 		}
 
-		static bool _is_red(node* n)
+		static bool _is_red(node_base* n)
 		{
 			return n != NULL && n->red;
 		}
 
-		static bool _is_black(node* n)
+		static bool _is_black(node_base* n)
 		{
 			return n == NULL || !n->red;
 		}
 
-		void _rotate_left(node* x)
+		void _rotate_left(node_base* x)
 		{
-			node* y = x->right;
+			node_base* y = x->right;
 			x->right = y->left;
 			if (y->left)
 				y->left->parent = x;
 			y->parent = x->parent;
-			if (x->parent == NULL)
-				_root = y;
+			if (x->parent->is_header)
+				_header.parent = y;
 			else if (x == x->parent->left)
 				x->parent->left = y;
 			else
@@ -557,15 +621,15 @@ namespace ft
 			x->parent = y;
 		}
 
-		void _rotate_right(node* x)
+		void _rotate_right(node_base* x)
 		{
-			node* y = x->left;
+			node_base* y = x->left;
 			x->left = y->right;
 			if (y->right)
 				y->right->parent = x;
 			y->parent = x->parent;
-			if (x->parent == NULL)
-				_root = y;
+			if (x->parent->is_header)
+				_header.parent = y;
 			else if (x == x->parent->right)
 				x->parent->right = y;
 			else
@@ -574,13 +638,13 @@ namespace ft
 			x->parent = y;
 		}
 
-		void _insert_fixup(node* z)
+		void _insert_fixup(node_base* z)
 		{
-			while (z->parent && _is_red(z->parent))
+			while (!z->parent->is_header && _is_red(z->parent))
 			{
 				if (z->parent == z->parent->parent->left)
 				{
-					node* uncle = z->parent->parent->right;
+					node_base* uncle = z->parent->parent->right;
 					if (_is_red(uncle))
 					{
 						z->parent->red = false;
@@ -602,7 +666,7 @@ namespace ft
 				}
 				else
 				{
-					node* uncle = z->parent->parent->left;
+					node_base* uncle = z->parent->parent->left;
 					if (_is_red(uncle))
 					{
 						z->parent->red = false;
@@ -623,18 +687,18 @@ namespace ft
 					}
 				}
 			}
-			if (_root)
-				_root->red = false;
+			if (_root())
+				_root()->red = false;
 		}
 
-		node* _find_node(const key_type& key) const
+		node_base* _find_node(const key_type& key) const
 		{
-			node* cur = _root;
+			node_base* cur = _root();
 			while (cur)
 			{
-				if (_comp(key, cur->value.first))
+				if (_comp(key, _value(cur).first))
 					cur = cur->left;
-				else if (_comp(cur->value.first, key))
+				else if (_comp(_value(cur).first, key))
 					cur = cur->right;
 				else
 					return cur;
@@ -642,13 +706,13 @@ namespace ft
 			return NULL;
 		}
 
-		node* _lower_bound_node(const key_type& key) const
+		node_base* _lower_bound_node(const key_type& key) const
 		{
-			node* cur = _root;
-			node* result = NULL;
+			node_base* cur = _root();
+			node_base* result = NULL;
 			while (cur)
 			{
-				if (!_comp(cur->value.first, key))
+				if (!_comp(_value(cur).first, key))
 				{
 					result = cur;
 					cur = cur->left;
@@ -659,13 +723,13 @@ namespace ft
 			return result;
 		}
 
-		node* _upper_bound_node(const key_type& key) const
+		node_base* _upper_bound_node(const key_type& key) const
 		{
-			node* cur = _root;
-			node* result = NULL;
+			node_base* cur = _root();
+			node_base* result = NULL;
 			while (cur)
 			{
-				if (_comp(key, cur->value.first))
+				if (_comp(key, _value(cur).first))
 				{
 					result = cur;
 					cur = cur->left;
@@ -676,10 +740,10 @@ namespace ft
 			return result;
 		}
 
-		void _transplant(node* old_node, node* new_node)
+		void _transplant(node_base* old_node, node_base* new_node)
 		{
-			if (old_node->parent == NULL)
-				_root = new_node;
+			if (old_node->parent->is_header)
+				_header.parent = new_node;
 			else if (old_node == old_node->parent->left)
 				old_node->parent->left = new_node;
 			else
@@ -688,11 +752,11 @@ namespace ft
 				new_node->parent = old_node->parent;
 		}
 
-		void _erase_node(node* target)
+		void _erase_node(node_base* target)
 		{
-			node* moved = target;
-			node* fix = NULL;
-			node* fix_parent = NULL;
+			node_base* moved = target;
+			node_base* fix = NULL;
+			node_base* fix_parent = NULL;
 			bool moved_was_red = moved->red;
 			if (target->left == NULL)
 			{
@@ -733,17 +797,18 @@ namespace ft
 			--_size;
 			if (!moved_was_red)
 				_erase_fixup(fix, fix_parent);
-			if (_root)
-				_root->red = false;
+			if (_root())
+				_root()->red = false;
+			_refresh_header();
 		}
 
-		void _erase_fixup(node* x, node* parent)
+		void _erase_fixup(node_base* x, node_base* parent)
 		{
-			while (x != _root && _is_black(x))
+			while (x != _root() && _is_black(x))
 			{
 				if (x == (parent ? parent->left : NULL))
 				{
-					node* sibling = parent ? parent->right : NULL;
+					node_base* sibling = parent ? parent->right : NULL;
 					if (_is_red(sibling))
 					{
 						sibling->red = false;
@@ -780,13 +845,13 @@ namespace ft
 							sibling->right->red = false;
 						if (parent)
 							_rotate_left(parent);
-						x = _root;
+						x = _root();
 						parent = NULL;
 					}
 				}
 				else
 				{
-					node* sibling = parent ? parent->left : NULL;
+					node_base* sibling = parent ? parent->left : NULL;
 					if (_is_red(sibling))
 					{
 						sibling->red = false;
@@ -823,7 +888,7 @@ namespace ft
 							sibling->left->red = false;
 						if (parent)
 							_rotate_right(parent);
-						x = _root;
+						x = _root();
 						parent = NULL;
 					}
 				}
@@ -832,7 +897,7 @@ namespace ft
 				x->red = false;
 		}
 
-		void _clear(node* n)
+		void _clear(node_base* n)
 		{
 			if (n == NULL)
 				return;


## `fix(map): 비교자 교환 실패 전에 tree 소유권 유지`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 87c690b..8c1134f 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -399,11 +399,11 @@ namespace ft
 
 		void swap(map& other)
 		{
+			std::swap(_comp, other._comp);
 			std::swap(_alloc, other._alloc);
 			std::swap(_node_alloc, other._node_alloc);
 			std::swap(_header.parent, other._header.parent);
 			std::swap(_size, other._size);
-			std::swap(_comp, other._comp);
 			_refresh_header();
 			other._refresh_header();
 		}
@@ -476,9 +476,9 @@ namespace ft
 
 		void _swap_tree_and_compare(map& other)
 		{
+			std::swap(_comp, other._comp);
 			std::swap(_header.parent, other._header.parent);
 			std::swap(_size, other._size);
-			std::swap(_comp, other._comp);
 			_refresh_header();
 			other._refresh_header();
 		}


