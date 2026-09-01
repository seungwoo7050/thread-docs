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


## `test(map): 범위 삭제 후 상태 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 03ecabf..737bbee 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -277,6 +277,11 @@ namespace
 			"map const begin");
 		require(ftconst.rbegin()->first == stdconst.rbegin()->first,
 			"map const rbegin");
+
+		ftcopy.erase(ftcopy.begin(), ftcopy.end());
+		stdcopy.erase(stdcopy.begin(), stdcopy.end());
+		require(ftcopy.empty() == stdcopy.empty(), "map range erase empty");
+		require(ftcopy.size() == stdcopy.size(), "map range erase size");
 	}
 
 	void test_map_stress_ordering()


## `test(map): 비교 함수 접근자 검증`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 737bbee..6d3ebfd 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -251,6 +251,11 @@ namespace
 			"map lower_bound gap");
 		require(ftm.upper_bound(13)->first == stdm.upper_bound(13)->first,
 			"map upper_bound near end");
+		require(ftm.key_comp()(1, 2) == stdm.key_comp()(1, 2),
+			"map key_comp");
+		require(ftm.value_comp()(*ftm.begin(), *ftm.upper_bound(1))
+				== stdm.value_comp()(*stdm.begin(), *stdm.upper_bound(1)),
+			"map value_comp");
 
 		ft::map<int, std::string>::reverse_iterator fmrit = ftm.rbegin();
 		std::map<int, std::string>::reverse_iterator smrit = stdm.rbegin();
