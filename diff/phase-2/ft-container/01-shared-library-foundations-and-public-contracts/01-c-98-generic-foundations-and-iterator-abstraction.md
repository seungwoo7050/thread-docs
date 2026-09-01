# C++98 제네릭 기반과 반복자 추상화

## `feat(type-traits): CXX98 타입 선택 도구 구현`

diff --git a/include/ft_type_traits.hpp b/include/ft_type_traits.hpp
new file mode 100644
index 0000000..43ab8d9
--- /dev/null
+++ b/include/ft_type_traits.hpp
@@ -0,0 +1,90 @@
+#ifndef FT_TYPE_TRAITS_HPP
+# define FT_TYPE_TRAITS_HPP
+
+namespace ft
+{
+	template <bool Cond, class T = void>
+	struct enable_if
+	{
+	};
+
+	template <class T>
+	struct enable_if<true, T>
+	{
+		typedef T type;
+	};
+
+	template <class T, T V>
+	struct integral_constant
+	{
+		static const T value = V;
+		typedef T value_type;
+		typedef integral_constant<T, V> type;
+		operator value_type() const { return value; }
+	};
+
+	typedef integral_constant<bool, true> true_type;
+	typedef integral_constant<bool, false> false_type;
+
+	template <class T>
+	struct is_integral : false_type
+	{
+	};
+
+	template <>
+	struct is_integral<bool> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<char> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<signed char> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<unsigned char> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<wchar_t> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<short> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<unsigned short> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<int> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<unsigned int> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<long> : true_type
+	{
+	};
+
+	template <>
+	struct is_integral<unsigned long> : true_type
+	{
+	};
+}
+
+#endif


## `feat(pair): 값 쌍과 관계 연산 구현`

diff --git a/include/ft_pair.hpp b/include/ft_pair.hpp
new file mode 100644
index 0000000..2606103
--- /dev/null
+++ b/include/ft_pair.hpp
@@ -0,0 +1,83 @@
+#ifndef FT_PAIR_HPP
+# define FT_PAIR_HPP
+
+namespace ft
+{
+	template <class T1, class T2>
+	struct pair
+	{
+		typedef T1 first_type;
+		typedef T2 second_type;
+
+		first_type first;
+		second_type second;
+
+		pair() : first(), second()
+		{
+		}
+
+		pair(const first_type& a, const second_type& b) : first(a), second(b)
+		{
+		}
+
+		template <class U, class V>
+		pair(const pair<U, V>& other) : first(other.first), second(other.second)
+		{
+		}
+
+		pair& operator=(const pair& other)
+		{
+			if (this != &other)
+			{
+				first = other.first;
+				second = other.second;
+			}
+			return *this;
+		}
+	};
+
+	template <class T1, class T2>
+	bool operator==(const pair<T1, T2>& lhs, const pair<T1, T2>& rhs)
+	{
+		return lhs.first == rhs.first && lhs.second == rhs.second;
+	}
+
+	template <class T1, class T2>
+	bool operator!=(const pair<T1, T2>& lhs, const pair<T1, T2>& rhs)
+	{
+		return !(lhs == rhs);
+	}
+
+	template <class T1, class T2>
+	bool operator<(const pair<T1, T2>& lhs, const pair<T1, T2>& rhs)
+	{
+		return lhs.first < rhs.first
+			|| (!(rhs.first < lhs.first) && lhs.second < rhs.second);
+	}
+
+	template <class T1, class T2>
+	bool operator<=(const pair<T1, T2>& lhs, const pair<T1, T2>& rhs)
+	{
+		return !(rhs < lhs);
+	}
+
+	template <class T1, class T2>
+	bool operator>(const pair<T1, T2>& lhs, const pair<T1, T2>& rhs)
+	{
+		return rhs < lhs;
+	}
+
+	template <class T1, class T2>
+	bool operator>=(const pair<T1, T2>& lhs, const pair<T1, T2>& rhs)
+	{
+		return !(lhs < rhs);
+	}
+
+	template <class T1, class T2>
+	pair<T1, T2> make_pair(T1 first, T2 second)
+	{
+		return pair<T1, T2>(first, second);
+	}
+}
+
+#endif


## `feat(algorithm): 공용 범위 비교 알고리즘 구현`

diff --git a/include/ft_algorithm.hpp b/include/ft_algorithm.hpp
new file mode 100644
index 0000000..bb15999
--- /dev/null
+++ b/include/ft_algorithm.hpp
@@ -0,0 +1,66 @@
+#ifndef FT_ALGORITHM_HPP
+# define FT_ALGORITHM_HPP
+
+namespace ft
+{
+	template <class InputIt1, class InputIt2>
+	bool equal(InputIt1 first1, InputIt1 last1, InputIt2 first2)
+	{
+		while (first1 != last1)
+		{
+			if (!(*first1 == *first2))
+				return false;
+			++first1;
+			++first2;
+		}
+		return true;
+	}
+
+	template <class InputIt1, class InputIt2, class BinaryPredicate>
+	bool equal(InputIt1 first1, InputIt1 last1, InputIt2 first2,
+		BinaryPredicate pred)
+	{
+		while (first1 != last1)
+		{
+			if (!pred(*first1, *first2))
+				return false;
+			++first1;
+			++first2;
+		}
+		return true;
+	}
+
+	template <class InputIt1, class InputIt2>
+	bool lexicographical_compare(InputIt1 first1, InputIt1 last1,
+		InputIt2 first2, InputIt2 last2)
+	{
+		while (first1 != last1 && first2 != last2)
+		{
+			if (*first1 < *first2)
+				return true;
+			if (*first2 < *first1)
+				return false;
+			++first1;
+			++first2;
+		}
+		return first1 == last1 && first2 != last2;
+	}
+
+	template <class InputIt1, class InputIt2, class Compare>
+	bool lexicographical_compare(InputIt1 first1, InputIt1 last1,
+		InputIt2 first2, InputIt2 last2, Compare comp)
+	{
+		while (first1 != last1 && first2 != last2)
+		{
+			if (comp(*first1, *first2))
+				return true;
+			if (comp(*first2, *first1))
+				return false;
+			++first1;
+			++first2;
+		}
+		return first1 == last1 && first2 != last2;
+	}
+}
+
+#endif


## `feat(iterator): iterator 기본 형식과 traits 정의`

diff --git a/include/ft_iterator.hpp b/include/ft_iterator.hpp
new file mode 100644
index 0000000..e722769
--- /dev/null
+++ b/include/ft_iterator.hpp
@@ -0,0 +1,51 @@
+#ifndef FT_ITERATOR_HPP
+# define FT_ITERATOR_HPP
+
+# include <cstddef>
+# include <iterator>
+
+namespace ft
+{
+	template <class Category, class T, class Distance = std::ptrdiff_t,
+		class Pointer = T*, class Reference = T&>
+	struct iterator
+	{
+		typedef Category iterator_category;
+		typedef T value_type;
+		typedef Distance difference_type;
+		typedef Pointer pointer;
+		typedef Reference reference;
+	};
+
+	template <class Iterator>
+	struct iterator_traits
+	{
+		typedef typename Iterator::difference_type difference_type;
+		typedef typename Iterator::value_type value_type;
+		typedef typename Iterator::pointer pointer;
+		typedef typename Iterator::reference reference;
+		typedef typename Iterator::iterator_category iterator_category;
+	};
+
+	template <class T>
+	struct iterator_traits<T*>
+	{
+		typedef std::ptrdiff_t difference_type;
+		typedef T value_type;
+		typedef T* pointer;
+		typedef T& reference;
+		typedef std::random_access_iterator_tag iterator_category;
+	};
+
+	template <class T>
+	struct iterator_traits<const T*>
+	{
+		typedef std::ptrdiff_t difference_type;
+		typedef T value_type;
+		typedef const T* pointer;
+		typedef const T& reference;
+		typedef std::random_access_iterator_tag iterator_category;
+	};
+}
+
+#endif


## `feat(iterator): 역방향 반복자의 양방향 동작 구현`

diff --git a/include/ft_iterator.hpp b/include/ft_iterator.hpp
index e722769..46cc9ac 100644
--- a/include/ft_iterator.hpp
+++ b/include/ft_iterator.hpp
@@ -46,6 +46,91 @@ namespace ft
 		typedef const T& reference;
 		typedef std::random_access_iterator_tag iterator_category;
 	};
+
+	template <class Iterator>
+	class reverse_iterator
+	{
+	public:
+		typedef Iterator iterator_type;
+		typedef typename iterator_traits<Iterator>::iterator_category iterator_category;
+		typedef typename iterator_traits<Iterator>::value_type value_type;
+		typedef typename iterator_traits<Iterator>::difference_type difference_type;
+		typedef typename iterator_traits<Iterator>::pointer pointer;
+		typedef typename iterator_traits<Iterator>::reference reference;
+
+		reverse_iterator() : _current()
+		{
+		}
+
+		explicit reverse_iterator(iterator_type it) : _current(it)
+		{
+		}
+
+		template <class U>
+		reverse_iterator(const reverse_iterator<U>& other)
+			: _current(other.base())
+		{
+		}
+
+		iterator_type base() const
+		{
+			return _current;
+		}
+
+		reference operator*() const
+		{
+			iterator_type tmp(_current);
+			return *--tmp;
+		}
+
+		pointer operator->() const
+		{
+			return &(operator*());
+		}
+
+		reverse_iterator& operator++()
+		{
+			--_current;
+			return *this;
+		}
+
+		reverse_iterator operator++(int)
+		{
+			reverse_iterator tmp(*this);
+			--_current;
+			return tmp;
+		}
+
+		reverse_iterator& operator--()
+		{
+			++_current;
+			return *this;
+		}
+
+		reverse_iterator operator--(int)
+		{
+			reverse_iterator tmp(*this);
+			++_current;
+			return tmp;
+		}
+
+	private:
+		iterator_type _current;
+	};
+
+	template <class It1, class It2>
+	bool operator==(const reverse_iterator<It1>& lhs,
+		const reverse_iterator<It2>& rhs)
+	{
+		return lhs.base() == rhs.base();
+	}
+
+	template <class It1, class It2>
+	bool operator!=(const reverse_iterator<It1>& lhs,
+		const reverse_iterator<It2>& rhs)
+	{
+		return !(lhs == rhs);
+	}
 }
 
 #endif


## `feat(iterator): 역방향 반복자의 임의 접근 연산 완성`

diff --git a/include/ft_iterator.hpp b/include/ft_iterator.hpp
index 46cc9ac..10f928b 100644
--- a/include/ft_iterator.hpp
+++ b/include/ft_iterator.hpp
@@ -114,6 +114,33 @@ namespace ft
 			return tmp;
 		}
 
+		reverse_iterator operator+(difference_type n) const
+		{
+			return reverse_iterator(_current - n);
+		}
+
+		reverse_iterator& operator+=(difference_type n)
+		{
+			_current -= n;
+			return *this;
+		}
+
+		reverse_iterator operator-(difference_type n) const
+		{
+			return reverse_iterator(_current + n);
+		}
+
+		reverse_iterator& operator-=(difference_type n)
+		{
+			_current += n;
+			return *this;
+		}
+
+		reference operator[](difference_type n) const
+		{
+			return *(*this + n);
+		}
+
 	private:
 		iterator_type _current;
 	};
@@ -131,6 +158,49 @@ namespace ft
 	{
 		return !(lhs == rhs);
 	}
+
+	template <class It1, class It2>
+	bool operator<(const reverse_iterator<It1>& lhs,
+		const reverse_iterator<It2>& rhs)
+	{
+		return rhs.base() < lhs.base();
+	}
+
+	template <class It1, class It2>
+	bool operator<=(const reverse_iterator<It1>& lhs,
+		const reverse_iterator<It2>& rhs)
+	{
+		return !(rhs < lhs);
+	}
+
+	template <class It1, class It2>
+	bool operator>(const reverse_iterator<It1>& lhs,
+		const reverse_iterator<It2>& rhs)
+	{
+		return rhs < lhs;
+	}
+
+	template <class It1, class It2>
+	bool operator>=(const reverse_iterator<It1>& lhs,
+		const reverse_iterator<It2>& rhs)
+	{
+		return !(lhs < rhs);
+	}
+
+	template <class It1, class It2>
+	typename reverse_iterator<It1>::difference_type operator-(
+		const reverse_iterator<It1>& lhs, const reverse_iterator<It2>& rhs)
+	{
+		return rhs.base() - lhs.base();
+	}
+
+	template <class Iterator>
+	reverse_iterator<Iterator> operator+(
+		typename reverse_iterator<Iterator>::difference_type n,
+		const reverse_iterator<Iterator>& it)
+	{
+		return it + n;
+	}
 }
 
 #endif


## `test(utils): 공용 타입·값·범위·반복자 도구 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
new file mode 100644
index 0000000..74f016e
--- /dev/null
+++ b/tests/test_containers.cpp
@@ -0,0 +1,46 @@
+#include <cstdlib>
+#include <iostream>
+#include <string>
+
+#include "ft_algorithm.hpp"
+#include "ft_iterator.hpp"
+#include "ft_pair.hpp"
+#include "ft_type_traits.hpp"
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
+	void test_utilities()
+	{
+		ft::pair<int, std::string> p = ft::make_pair(3, std::string("three"));
+		require(p.first == 3 && p.second == "three", "make_pair value");
+		require(ft::is_integral<int>::value, "is_integral int");
+		require(!ft::is_integral<std::string>::value, "is_integral string");
+
+		int a[] = {1, 2, 3};
+		int b[] = {1, 2, 4};
+		require(ft::equal(a, a + 2, b), "equal prefix");
+		require(ft::lexicographical_compare(a, a + 3, b, b + 3),
+			"lexicographical_compare arrays");
+
+		ft::reverse_iterator<int*> reverse(a + 3);
+		require(*reverse == 3, "reverse_iterator dereference");
+		require(ft::iterator_traits<int*>::difference_type(3) == 3,
+			"iterator_traits difference type");
+	}
+}
+
+int main()
+{
+	test_utilities();
+	std::cout << "ft_containers checks passed" << std::endl;
+	return 0;
+}
