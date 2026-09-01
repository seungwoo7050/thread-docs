## `test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성`

diff --git a/tests/test_containers.cpp b/tests/test_containers.cpp
index 6d3ebfd..5e4628b 100644
--- a/tests/test_containers.cpp
+++ b/tests/test_containers.cpp
@@ -77,15 +77,24 @@ namespace
 		require(ftcopy == ftv, "vector equality");
 		require(!(ftcopy < ftv), "vector less equal case");
 
-		std::vector<int> insert_source(stdcopy.begin(), stdcopy.begin() + 4);
+		std::vector<int> insert_expected;
+		insert_expected.reserve(stdcopy.size() + 4);
+		for (std::size_t i = 0; i < 3; ++i)
+			insert_expected.push_back(stdcopy[i]);
+		for (std::size_t i = 0; i < 4; ++i)
+			insert_expected.push_back(stdcopy[i]);
+		for (std::size_t i = 3; i < stdcopy.size(); ++i)
+			insert_expected.push_back(stdcopy[i]);
 		ftcopy.insert(ftcopy.begin() + 3, ftcopy.begin(), ftcopy.begin() + 4);
-		stdcopy.insert(stdcopy.begin() + 3,
-			insert_source.begin(), insert_source.end());
+		stdcopy.swap(insert_expected);
 		compare_vector(ftcopy, stdcopy, "self range insert");
 
-		std::vector<int> assign_source(stdcopy.begin() + 2, stdcopy.end() - 1);
+		std::vector<int> assign_expected;
+		assign_expected.reserve(stdcopy.size() - 3);
+		for (std::size_t i = 2; i + 1 < stdcopy.size(); ++i)
+			assign_expected.push_back(stdcopy[i]);
 		ftcopy.assign(ftcopy.begin() + 2, ftcopy.end() - 1);
-		stdcopy.assign(assign_source.begin(), assign_source.end());
+		stdcopy.swap(assign_expected);
 		compare_vector(ftcopy, stdcopy, "self range assign");
 
 		ft::vector<int>::reverse_iterator frit = ftcopy.rbegin();
