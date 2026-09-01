# 레드-블랙 균형화와 구조·복잡도 불변식

## `feat(map): 레드-블랙 삽입 회전과 색 보정 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index 710d7b7..b182b35 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -35,9 +35,10 @@ namespace ft
 			node* parent;
 			node* left;
 			node* right;
+			bool red;
 
 			explicit node(const value_type& v)
-				: value(v), parent(NULL), left(NULL), right(NULL)
+				: value(v), parent(NULL), left(NULL), right(NULL), red(true)
 			{
 			}
 		};
@@ -291,6 +292,7 @@ namespace ft
 			if (_root == NULL)
 			{
 				_root = _create_node(value);
+				_root->red = false;
 				++_size;
 				return ft::make_pair(iterator(_root, _root), true);
 			}
@@ -312,6 +314,7 @@ namespace ft
 				parent->left = created;
 			else
 				parent->right = created;
+			_insert_fixup(created);
 			++_size;
 			return ft::make_pair(iterator(created, _root), true);
 		}
@@ -498,6 +501,98 @@ namespace ft
 			return parent;
 		}
 
+		static bool _is_red(node* n)
+		{
+			return n != NULL && n->red;
+		}
+
+		void _rotate_left(node* x)
+		{
+			node* y = x->right;
+			x->right = y->left;
+			if (y->left)
+				y->left->parent = x;
+			y->parent = x->parent;
+			if (x->parent == NULL)
+				_root = y;
+			else if (x == x->parent->left)
+				x->parent->left = y;
+			else
+				x->parent->right = y;
+			y->left = x;
+			x->parent = y;
+		}
+
+		void _rotate_right(node* x)
+		{
+			node* y = x->left;
+			x->left = y->right;
+			if (y->right)
+				y->right->parent = x;
+			y->parent = x->parent;
+			if (x->parent == NULL)
+				_root = y;
+			else if (x == x->parent->right)
+				x->parent->right = y;
+			else
+				x->parent->left = y;
+			y->right = x;
+			x->parent = y;
+		}
+
+		void _insert_fixup(node* z)
+		{
+			while (z->parent && _is_red(z->parent))
+			{
+				if (z->parent == z->parent->parent->left)
+				{
+					node* uncle = z->parent->parent->right;
+					if (_is_red(uncle))
+					{
+						z->parent->red = false;
+						uncle->red = false;
+						z->parent->parent->red = true;
+						z = z->parent->parent;
+					}
+					else
+					{
+						if (z == z->parent->right)
+						{
+							z = z->parent;
+							_rotate_left(z);
+						}
+						z->parent->red = false;
+						z->parent->parent->red = true;
+						_rotate_right(z->parent->parent);
+					}
+				}
+				else
+				{
+					node* uncle = z->parent->parent->left;
+					if (_is_red(uncle))
+					{
+						z->parent->red = false;
+						uncle->red = false;
+						z->parent->parent->red = true;
+						z = z->parent->parent;
+					}
+					else
+					{
+						if (z == z->parent->left)
+						{
+							z = z->parent;
+							_rotate_right(z);
+						}
+						z->parent->red = false;
+						z->parent->parent->red = true;
+						_rotate_left(z->parent->parent);
+					}
+				}
+			}
+			if (_root)
+				_root->red = false;
+		}
+
 		node* _find_node(const key_type& key) const
 		{
 			node* cur = _root;


## `test(map): 정렬 입력 삽입과 검색 경계 stress 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 4ae588b..8e0a203 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -153,6 +153,69 @@ namespace
 		require(fit == ftm.end() && sit == stdm.end(), label + " end");
 	}
 
+	std::string map_value(int key)
+	{
+		std::ostringstream oss;
+		oss << "value-" << key;
+		return oss.str();
+	}
+
+	void insert_key(ft::map<int, std::string>& ftm,
+		std::map<int, std::string>& stdm, int key)
+	{
+		ft::pair<ft::map<int, std::string>::iterator, bool> fr =
+			ftm.insert(ft::make_pair(key, map_value(key)));
+		std::pair<std::map<int, std::string>::iterator, bool> sr =
+			stdm.insert(std::make_pair(key, map_value(key)));
+		require(fr.second == sr.second, "map insert result");
+		require(fr.first->first == sr.first->first, "map insert iterator key");
+	}
+
+	void check_map_queries(const ft::map<int, std::string>& ftm,
+		const std::map<int, std::string>& stdm, const std::string& label)
+	{
+		for (int key = -3; key <= 135; key += 7)
+		{
+			ft::map<int, std::string>::const_iterator fit = ftm.find(key);
+			std::map<int, std::string>::const_iterator sit = stdm.find(key);
+			require((fit == ftm.end()) == (sit == stdm.end()),
+				label + " find presence");
+			if (sit != stdm.end())
+				require(fit->second == sit->second, label + " find value");
+
+			fit = ftm.lower_bound(key);
+			sit = stdm.lower_bound(key);
+			require((fit == ftm.end()) == (sit == stdm.end()),
+				label + " lower_bound presence");
+			if (sit != stdm.end())
+				require(fit->first == sit->first, label + " lower_bound key");
+
+			fit = ftm.upper_bound(key);
+			sit = stdm.upper_bound(key);
+			require((fit == ftm.end()) == (sit == stdm.end()),
+				label + " upper_bound presence");
+			if (sit != stdm.end())
+				require(fit->first == sit->first, label + " upper_bound key");
+
+			ft::pair<ft::map<int, std::string>::const_iterator,
+				ft::map<int, std::string>::const_iterator> fr =
+				ftm.equal_range(key);
+			std::pair<std::map<int, std::string>::const_iterator,
+				std::map<int, std::string>::const_iterator> sr =
+				stdm.equal_range(key);
+			require((fr.first == ftm.end()) == (sr.first == stdm.end()),
+				label + " equal_range first presence");
+			require((fr.second == ftm.end()) == (sr.second == stdm.end()),
+				label + " equal_range second presence");
+			if (sr.first != stdm.end())
+				require(fr.first->first == sr.first->first,
+					label + " equal_range first key");
+			if (sr.second != stdm.end())
+				require(fr.second->first == sr.second->first,
+					label + " equal_range second key");
+		}
+	}
+
 	void test_map()
 	{
 		ft::map<int, std::string> ftm;
@@ -215,6 +278,23 @@ namespace
 		require(ftconst.rbegin()->first == stdconst.rbegin()->first,
 			"map const rbegin");
 	}
+
+	void test_map_stress_ordering()
+	{
+		ft::map<int, std::string> ftasc;
+		std::map<int, std::string> stdasc;
+		for (int i = 1; i <= 96; ++i)
+			insert_key(ftasc, stdasc, i);
+		compare_map(ftasc, stdasc, "map ascending insert");
+		check_map_queries(ftasc, stdasc, "map ascending queries");
+
+		ft::map<int, std::string> ftdesc;
+		std::map<int, std::string> stddesc;
+		for (int i = 96; i >= 1; --i)
+			insert_key(ftdesc, stddesc, i);
+		compare_map(ftdesc, stddesc, "map descending insert");
+		check_map_queries(ftdesc, stddesc, "map descending queries");
+	}
 }
 
 int main()
@@ -223,6 +303,7 @@ int main()
 	test_vector();
 	test_stack();
 	test_map();
+	test_map_stress_ordering();
 	std::cout << "ft_containers checks passed" << std::endl;
 	return 0;
 }


## `feat(map): 레드-블랙 삭제 보정 구현`

diff --git a/include/ft_map.hpp b/include/ft_map.hpp
index b182b35..0f0f986 100644
--- a/include/ft_map.hpp
+++ b/include/ft_map.hpp
@@ -506,6 +506,11 @@ namespace ft
 			return n != NULL && n->red;
 		}
 
+		static bool _is_black(node* n)
+		{
+			return n == NULL || !n->red;
+		}
+
 		void _rotate_left(node* x)
 		{
 			node* y = x->right;
@@ -656,25 +661,146 @@ namespace ft
 
 		void _erase_node(node* target)
 		{
+			node* moved = target;
+			node* fix = NULL;
+			node* fix_parent = NULL;
+			bool moved_was_red = moved->red;
 			if (target->left == NULL)
+			{
+				fix = target->right;
+				fix_parent = target->parent;
 				_transplant(target, target->right);
+			}
 			else if (target->right == NULL)
+			{
+				fix = target->left;
+				fix_parent = target->parent;
 				_transplant(target, target->left);
+			}
 			else
 			{
-				node* successor = _minimum(target->right);
-				if (successor->parent != target)
+				moved = _minimum(target->right);
+				moved_was_red = moved->red;
+				fix = moved->right;
+				if (moved->parent == target)
+				{
+					fix_parent = moved;
+					if (fix)
+						fix->parent = moved;
+				}
+				else
 				{
-					_transplant(successor, successor->right);
-					successor->right = target->right;
-					successor->right->parent = successor;
+					fix_parent = moved->parent;
+					_transplant(moved, moved->right);
+					moved->right = target->right;
+					moved->right->parent = moved;
 				}
-				_transplant(target, successor);
-				successor->left = target->left;
-				successor->left->parent = successor;
+				_transplant(target, moved);
+				moved->left = target->left;
+				moved->left->parent = moved;
+				moved->red = target->red;
 			}
 			_destroy_node(target);
 			--_size;
+			if (!moved_was_red)
+				_erase_fixup(fix, fix_parent);
+			if (_root)
+				_root->red = false;
+		}
+
+		void _erase_fixup(node* x, node* parent)
+		{
+			while (x != _root && _is_black(x))
+			{
+				if (x == (parent ? parent->left : NULL))
+				{
+					node* sibling = parent ? parent->right : NULL;
+					if (_is_red(sibling))
+					{
+						sibling->red = false;
+						parent->red = true;
+						_rotate_left(parent);
+						sibling = parent->right;
+					}
+					if (_is_black(sibling ? sibling->left : NULL)
+						&& _is_black(sibling ? sibling->right : NULL))
+					{
+						if (sibling)
+							sibling->red = true;
+						x = parent;
+						parent = x ? x->parent : NULL;
+					}
+					else
+					{
+						if (_is_black(sibling ? sibling->right : NULL))
+						{
+							if (sibling && sibling->left)
+								sibling->left->red = false;
+							if (sibling)
+							{
+								sibling->red = true;
+								_rotate_right(sibling);
+							}
+							sibling = parent ? parent->right : NULL;
+						}
+						if (sibling)
+							sibling->red = parent ? parent->red : false;
+						if (parent)
+							parent->red = false;
+						if (sibling && sibling->right)
+							sibling->right->red = false;
+						if (parent)
+							_rotate_left(parent);
+						x = _root;
+						parent = NULL;
+					}
+				}
+				else
+				{
+					node* sibling = parent ? parent->left : NULL;
+					if (_is_red(sibling))
+					{
+						sibling->red = false;
+						parent->red = true;
+						_rotate_right(parent);
+						sibling = parent->left;
+					}
+					if (_is_black(sibling ? sibling->right : NULL)
+						&& _is_black(sibling ? sibling->left : NULL))
+					{
+						if (sibling)
+							sibling->red = true;
+						x = parent;
+						parent = x ? x->parent : NULL;
+					}
+					else
+					{
+						if (_is_black(sibling ? sibling->left : NULL))
+						{
+							if (sibling && sibling->right)
+								sibling->right->red = false;
+							if (sibling)
+							{
+								sibling->red = true;
+								_rotate_left(sibling);
+							}
+							sibling = parent ? parent->left : NULL;
+						}
+						if (sibling)
+							sibling->red = parent ? parent->red : false;
+						if (parent)
+							parent->red = false;
+						if (sibling && sibling->left)
+							sibling->left->red = false;
+						if (parent)
+							_rotate_right(parent);
+						x = _root;
+						parent = NULL;
+					}
+				}
+			}
+			if (x)
+				x->red = false;
 		}
 
 		void _clear(node* n)


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


