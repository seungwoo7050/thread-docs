# 개발자 기술면접 워크북 — 마스터 인덱스
현재 프로젝트에 축적된 9개 DevThread의 커밋 기록에서 실제로 확인되는 내용만 사용했다. 프로젝트는 C++98 헤더 전용 컨테이너 라이브러리이며, 확인된 핵심 범위는 제네릭 도구, `vector`, `stack`, 비교자 기반 `map`, 레드-블랙 트리, allocator·객체 수명·예외 경로, 반복자 안정성, 구조·복잡도 검증, 공개 헤더 소비자 빌드다.

I/O·네트워크·동시성·비동기 처리와 별도의 보안 경계는 확인된 구현 주제가 아니므로 면접 항목을 억지로 만들지 않았다. S/A 행의 점 ID가 상세 워크북과 완전성 검증의 기준이다.
## 우선순위 기준
| 등급 | 의미 |
| --- | --- |
| S | 반드시 준비. 질문과 직접 구현 모두 가치가 높음 |
| A | 준비 가치가 높음. 질문 또는 핵심 구현 가능성이 높음 |
| B | 구현보다 설계·개념 설명 준비가 중요함 |
| C | 별도 면접 준비 항목으로 만들 필요가 낮음 |

## 전체 Thread/커밋 선별 결과

### Thread 01 — C++98 제네릭 기반과 반복자 추상화
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A · GEN-01 | 01 | `feat(type-traits): CXX98 타입 선택 도구 구현` | `include/ft_type_traits.hpp`; `enable_if`, `integral_constant`, `is_integral` | C++98 SFINAE와 count/range 오버로드 분리 | 현대 type traits 없이 공개 API의 모호성을 제거하는 일반화 가능한 언어 기본기다. | 높음 | 높음 | 04·05 |
| C | 01 | `feat(pair): 값 쌍과 관계 연산 구현` | `include/ft_pair.hpp`; `pair`, `make_pair`, 관계 연산자 | 값 쌍과 사전식 관계 연산 | 필요한 기반 도구지만 구현이 작고 다른 핵심 invariant를 거의 드러내지 않는다. | 낮음 | 낮음 | 02·06 |
| B | 01 | `feat(algorithm): 공용 범위 비교 알고리즘 구현` | `include/ft_algorithm.hpp`; `equal`, `lexicographical_compare` | 범위 비교와 사전식 순서 | 컨테이너 관계 연산의 기반이지만 직접 구현 난도보다 계약 설명 가치가 크다. | 중간 | 낮음 | 02·04·06 |
| B | 01 | `feat(iterator): iterator 기본 형식과 traits 정의` | `include/ft_iterator.hpp`; `iterator`, `iterator_traits` | 반복자 traits와 포인터 특수화 | 제네릭 알고리즘의 형식 추출 원리를 묻기 좋지만 프로젝트의 고난도 지점은 아니다. | 중간 | 낮음 | 04·06·08 |
| A · GEN-02 | 01 | `feat(iterator): 역방향 반복자의 양방향 동작 구현` | `include/ft_iterator.hpp`; `reverse_iterator`, `base`, `operator*`, `operator++`, `operator--` | one-past base invariant와 방향 반전 | off-by-one과 증감 방향을 동시에 검증하는 대표 반복자 문제다. | 높음 | 높음 | 04·06·08 |
| A · GEN-02 | 01 | `feat(iterator): 역방향 반복자의 임의 접근 연산 완성` | `include/ft_iterator.hpp`; `operator+`, `operator-`, `operator[]`, 혼합 비교 | 역방향 임의 접근 산술과 순서 반전 | 기본 invariant를 산술·비교까지 일관되게 확장하는 능력을 확인할 수 있어 GEN-02에 통합한다. | 높음 | 중간 | 04·08 |
| C | 01 | `test(utils): 공용 타입·값·범위·반복자 도구 검증` | `tests/test_containers.cpp` | 기반 유틸리티 공개 동작 확인 | 기능 확인 성격이 강하고 독립적인 면접 문제로 만들 추가 판단이 적다. | 낮음 | 부적합 | 01 |

### Thread 02 — 컨테이너 조합과 헤더 전용 공개 계약
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| C | 02 | `docs(readme): C++98 컨테이너 개발 기준 정의` | `README.md` | 프로젝트 범위와 개발 기준 | 프로젝트 설명에는 필요하지만 기술 역량을 직접 검증하는 항목은 아니다. | 낮음 | 부적합 | 01·03 |
| C | 02 | `feat(stack): vector 기반 stack 어댑터 구현` | `include/ft_stack.hpp`; `stack` | sequence adapter 위임 | 대부분 하위 컨테이너에 위임하는 얇은 boilerplate다. | 낮음 | 낮음 | 04 |
| C | 02 | `test(stack): 기본 동작과 관계 연산 검증` | `tests/test_containers.cpp`; `test_stack` | stack 공개 결과 비교 | 반복적인 공개 결과 확인으로 별도 면접 준비 효율이 낮다. | 낮음 | 부적합 | 02 |
| C | 02 | `feat(headers): 공용 도구와 순차 컨테이너 통합 헤더 추가` | `include/ft_containers.hpp` | 통합 공개 헤더 구성 | include 목록을 모으는 설정성 변경에 가깝다. | 낮음 | 부적합 | 03 |
| C | 02 | `test(headers): 통합 헤더의 순차 컨테이너 표면 검증` | `tests/test_containers.cpp` | 통합 헤더 공개 표면 확인 | 핵심 구현보다 include 경로 확인에 가깝다. | 낮음 | 부적합 | 03 |
| C | 02 | `feat(map): 관계 연산과 통합 공개 헤더 완성` | `include/ft_map.hpp`, `include/ft_containers.hpp`; `value_compare`, map 관계 연산 | 공개 조합과 관계 연산 | 사전식 비교 재사용이 중심이며 map 내부 구조보다 면접 가치가 낮다. | 낮음 | 낮음 | 01·06 |
| B | 02 | `test(headers): 공개 헤더를 각각 독립 compile` | `tests/headers/*.cpp`, `Makefile`; `headers` target | 헤더 self-containment | 헤더 전용 라이브러리 경계를 설명하기 좋은 빌드 계약이지만 백지 구현 문제는 아니다. | 중간 | 부적합 | 03 |
| B | 02 | `test(consumer): 다중 번역 단위 공개 헤더 사용 검증` | `tests/consumer/consumer_api.hpp`, `main.cpp`, `map_consumer.cpp`, `vector_consumer.cpp`; `consumer` target | ODR·링크·외부 소비자 경계 | 단일 테스트 번역 단위가 놓치는 링크 경계를 설명할 가치가 있다. | 중간 | 부적합 | 03 |
| C | 02 | `docs: improve README with project visuals` | `README.md`, `docs/images/container-invariants.svg` | 프로젝트 표현 invariant 문서화 | 학습 보조 자료로는 유용하지만 별도 기술면접 항목으로 만들 필요는 낮다. | 낮음 | 부적합 | 04·07·08·09 |

### Thread 03 — 다층 검증 게이트와 크로스플랫폼 CI
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| B | 03 | `build(makefile): CXX98 검사 빌드 구성` | `Makefile`; `test`, `clean`, `fclean`, `re` | 엄격 C++98 빌드 진입점 | 재현 가능한 기본 검증 게이트의 역할을 설명할 수 있어야 하지만 구현 문제는 아니다. | 중간 | 부적합 | 02 |
| B | 03 | `test(headers): 공개 헤더를 각각 독립 compile` | `Makefile`, `tests/headers/*.cpp`; `headers` target | 헤더별 독립 컴파일 | 전이 include 의존을 탐지하는 헤더 전용 배포 경계다. | 중간 | 부적합 | 02 |
| B | 03 | `test(consumer): 다중 번역 단위 공개 헤더 사용 검증` | `Makefile`, `tests/consumer/*`; `consumer`, `check` target | 다중 번역 단위 소비자 검증 | ODR와 링크 단계 문제를 별도 계층에서 확인하는 설계 판단이 핵심이다. | 중간 | 부적합 | 02 |
| B | 03 | `build(makefile): 격리된 sanitizer 검사 대상 추가` | `Makefile`; `sanitize` target, 분리된 build 경로 | sanitizer 빌드 격리 | 일반 빌드 산출물과 계측 빌드를 섞지 않는 검증 환경 설계가 중요하다. | 중간 | 부적합 | 04·05·07·09 |
| B | 03 | `ci: compiler 행렬과 sanitizer 검사 구성` | `.github/workflows/ci.yml` | 컴파일러·sanitizer 행렬 | 도구별 차이와 메모리 오류를 CI 게이트로 분리하는 설명 가치가 있다. | 중간 | 부적합 | 02·05·07·09 |
| B | 03 | `ci: harden cross-platform verification` | `.github/workflows/cpp-ft-container-ci.yml` | 크로스플랫폼 검증 강화 | 플랫폼 차이로 인한 우연한 성공을 줄이는 운영 판단이지만 직접 구현 면접과는 거리가 있다. | 중간 | 부적합 | 02·03 |

### Thread 04 — allocator 기반 vector 저장 모델과 시퀀스 API
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| S · VEC-01 | 04 | `feat(vector): allocator 기반 저장소 수명 관리` | `include/ft_vector.hpp`; `_alloc`, `_data`, `_size`, `_capacity`, 생성자, 소멸자, `_destroy_storage` | 원시 저장소와 생성된 객체 수명 invariant | C++ 객체 수명·allocator·RAII를 한 번에 검증하는 프로젝트 핵심이다. | 높음 | 높음 | 05 |
| B | 04 | `feat(vector): 반복자와 원소 접근 경계 구현` | `include/ft_vector.hpp`; `begin`, `end`, `at`, `operator[]`, `front`, `back` | 포인터 반복자와 접근 사전 조건 | 필수 API지만 대부분 경계 계약 설명이며 더 대표적인 빈 저장소 문제는 VEC-04가 맡는다. | 중간 | 낮음 | 01·05 |
| S · VEC-02 | 04 | `feat(vector): 용량 확장과 원소 재배치 구현` | `include/ft_vector.hpp`; `reserve`, `push_back`, `_reallocate`, `_next_capacity` | 재할당 트랜잭션과 소유권 commit | 새 블록 구축·rollback·기존 상태 교체 순서가 예외 안전성의 핵심이다. | 높음 | 높음 | 05 |
| A · GEN-01, VEC-06 | 04 | `feat(vector): 크기 변경과 값 범위 할당 구현` | `include/ft_vector.hpp`; `resize`, `assign`, 범위 생성자 | 범위 오버로드 선택과 크기 변경 rollback | SFINAE와 suffix 생성 실패 정리를 함께 다루며 Thread 05의 강화 구현에 통합한다. | 높음 | 높음 | 01·05 |
| A · VEC-07 | 04 | `feat(vector): 중간 변경 연산과 관계 비교 완성` | `include/ft_vector.hpp`; `insert`, `erase`, `clear`, `swap` | 중간 삽입의 이동·수명 경계 | 초기 구현 자체보다 Thread 05에서 드러난 제자리 삽입 객체 수명 문제가 대표적이다. | 높음 | 높음 | 05 |
| C | 04 | `test(vector): 핵심 공개 동작을 표준 결과와 비교` | `tests/test_containers.cpp`; `test_vector`, `compare_vector` | vector 공개 결과 차등 확인 | 정상 경로 회귀에는 유용하지만 독립 면접 지점으로는 반복적이다. | 낮음 | 부적합 | 04·05 |
| A · VEC-03 | 04 | `fix(vector): 용량 계산을 allocator 상한에서 포화` | `include/ft_vector.hpp`; `_next_capacity`, `resize`, `assign`, `insert` | overflow-safe 성장과 `max_size` 경계 | 부호 없는 산술 overflow와 allocator 상한을 동시에 다루는 실전 경계 문제다. | 높음 | 높음 | 05 |
| B | 04 | `test(vector): 제한 allocator에서 용량 상한 검증` | `tests/test_vector_exceptions.cpp`; `bounded_allocator`, `test_bounded_growth` | 작은 `max_size`로 성장 경계 노출 | 좋은 경계 테스트지만 구현 포인트는 VEC-03에 통합한다. | 중간 | 낮음 | 05 |
| C | 04 | `test(vector): 역방향 순회 결과 검증` | `tests/test_containers.cpp` | reverse iterator 조합 확인 | 핵심 구현은 Thread 01의 GEN-02가 대표한다. | 낮음 | 부적합 | 01 |
| A · VEC-04 | 04 | `fix(vector): allocator 형식과 빈 반복자 연산 보정` | `include/ft_vector.hpp`; allocator 기반 typedef, `_iterator_at`, `_index_of`, `begin`, `end`, `insert`, `erase` | 빈 저장소 포인터 산술과 allocator 공개 형식 | NULL 포인터 산술 UB와 allocator 계약을 동시에 드러내는 작은 고가치 경계다. | 높음 | 높음 | 05 |
| B | 04 | `test(vector): 빈 저장소와 allocator 상태 검증` | `tests/test_vector_exceptions.cpp` | 빈 반복자·stateful allocator 회귀 | VEC-04와 allocator 상태를 뒷받침하는 테스트로 상세 문제에 통합한다. | 중간 | 낮음 | 05·09 |

### Thread 05 — vector 객체 수명·별칭·예외 안전성
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A · GEN-01, VEC-06 | 05 | `feat(vector): 크기 변경과 값 범위 할당 구현` | `include/ft_vector.hpp`; `resize`, `assign`, 범위 API | 범위 선택과 변경 연산의 상태 전이 | Thread 04와 중복되는 역량으로 GEN-01·VEC-06 대표 문제에 통합한다. | 높음 | 높음 | 01·04 |
| A · VEC-07 | 05 | `feat(vector): 중간 변경 연산과 관계 비교 완성` | `include/ft_vector.hpp`; `insert`, `erase`, `swap` | 삽입·삭제 이동 경계 | 이후 수명·예외 수정의 출발점으로 VEC-07에 통합한다. | 높음 | 높음 | 04 |
| A · VEC-03 | 05 | `fix(vector): 용량 계산을 allocator 상한에서 포화` | `include/ft_vector.hpp`; `_next_capacity` | 포화 성장 산술 | Thread 04와 같은 대표 문제 VEC-03으로 묶는다. | 높음 | 높음 | 04 |
| B | 05 | `test(vector): 제한 allocator에서 용량 상한 검증` | `tests/test_vector_exceptions.cpp`; `bounded_allocator` | 용량 경계 실패 주입 | 구현 판단은 VEC-03, 테스트 일반화는 TEST-01이 대표한다. | 중간 | 낮음 | 04 |
| S · VEC-05 | 05 | `fix(vector): 자기 범위 assign과 insert 입력 보존` | `include/ft_vector.hpp`; 범위 `assign`, 범위 `insert`, 임시 `vector` snapshot | self-range와 입력 별칭 | 수정 중 입력 반복자가 무효화되는 비직관적 경계를 직접 다룬다. | 높음 | 높음 | 04 |
| B | 05 | `test(vector): 자기 범위 변경 결과 검증` | `tests/test_containers.cpp` | 자기 범위 정상 결과 | 대표 구현 문제 VEC-05를 뒷받침하는 회귀 테스트다. | 중간 | 낮음 | 05 |
| S · VEC-06 | 05 | `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리` | `include/ft_vector.hpp`; 복사 대입, fill `assign`, `resize`, `push_back` | 강한 보장과 suffix rollback | 부분 상태 노출을 피하는 commit/rollback 설계가 면접 가치가 매우 높다. | 높음 | 높음 | 04 |
| A · TEST-01 | 05 | `test(vector): 생성·대입·크기 변경 실패 주입` | `tests/test_vector_exceptions.cpp`; `tracked_value`, `tracking_allocator` | 객체 수명·할당 실패 주입 | 예외 안전성을 결정적으로 검증하는 일반화 가능한 테스트 설계다. | 높음 | 중간 | 09 |
| S · VEC-07 | 05 | `fix(vector): fill·range 삽입의 객체 수명 보존` | `include/ft_vector.hpp`; fill/range `insert`, 제자리·재할당 경로 | 미생성 저장소와 살아 있는 객체 이동 | placement construction과 assignment를 잘못 섞기 쉬운 대표 수명 문제다. | 높음 | 높음 | 04 |
| A · TEST-01, VEC-07 | 05 | `test(vector): 삽입 복사·대입·할당 실패 sweep` | `tests/test_vector_exceptions.cpp` | 삽입의 모든 실패 지점 sweep | VEC-07의 경계를 입증하면서 공통 실패 주입 문제 TEST-01로도 일반화된다. | 높음 | 중간 | 09 |
| B | 05 | `test(vector): 자기 범위 기대값을 명시적 snapshot으로 구성` | `tests/test_containers.cpp` | 테스트 oracle의 입력 별칭 제거 | 테스트 자체가 대상과 같은 별칭 버그를 갖지 않게 하는 좋은 판단이지만 별도 문제는 과하다. | 중간 | 낮음 | 05 |

### Thread 06 — 비교자 기반 ordered map 인터페이스와 lookup pipeline
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A · MAP-01 | 06 | `feat(map): 노드 소유권과 빈 tree 상태 구현` | `include/ft_map.hpp`; `node`, `node_allocator`, `_create_node`, `_destroy_node`, `_clear` | allocator로 관리하는 tree 노드의 소유권과 빈 상태 | 값 allocator를 노드 allocator로 재바인딩하고 노드 생성·소멸을 한 쌍으로 관리하는 기반 설계다. | 높음 | 중간 | 09 |
| B | 06 | `feat(map): 가변 반복자와 tree 순회 구현` | `include/ft_map.hpp`; `iterator`, `_minimum`, `_maximum`, `_next`, `_previous` | 부모 링크를 이용한 중위 순회 | 검색 트리 반복자의 핵심이지만 이후 header sentinel 표현으로 교체되어 최종 설계의 대표 지점은 Thread 08이다. | 높음 | 중간 | 08 |
| A · SEN-03 | 06 | `feat(map): 상수와 역방향 반복자 구현` | `include/ft_map.hpp`; `const_iterator`, `reverse_iterator`, `const_reverse_iterator` | const 전환과 양방향 반복자 위의 역방향 순회 | 가변 반복자에서 상수 반복자로의 안전한 전환과 reverse adapter의 경계를 함께 확인할 수 있다. | 높음 | 중간 | 01·08 |
| A · MTX-01, MTX-02 | 06 | `feat(map): 삽입과 첨자 및 복사 동작 구현` | `include/ft_map.hpp`; `insert`, `operator[]`, range/copy constructor, `operator=` | 삽입·복사 경로의 상태 전이와 실패 격리 | 노드 연결, 중복 판정, 기본 생성, 복사 소유권을 한 번에 다루며 Thread 09의 트랜잭션 개선으로 이어진다. | 높음 | 높음 | 09 |
| S · MAP-02 | 06 | `feat(map): 검색과 경계 query 구현` | `include/ft_map.hpp`; `_find_node`, `_lower_bound_node`, `_upper_bound_node`, `find`, `count`, `lower_bound`, `upper_bound`, `equal_range` | 비교자 등가성과 한 번의 tree 하강으로 구현하는 경계 질의 | 동등 연산자 없이 strict weak ordering만으로 검색·경계를 정의하는 자료구조 핵심 문제다. | 높음 | 높음 | 07 |
| A · MAP-03 | 06 | `feat(map): 삭제와 clear 및 swap 구현` | `include/ft_map.hpp`; erase overloads, `_transplant`, `_erase_node`, `clear`, `swap` | BST 삭제의 구조 변경과 다음 반복자 보존 | 0·1·2자식 삭제, successor 이식, 부모 링크와 size 정합성을 설명·구현하기 좋다. | 높음 | 높음 | 07·08 |
| C | 06 | `feat(map): 관계 연산과 통합 공개 헤더 완성` | `include/ft_map.hpp`, `include/ft_containers.hpp`; `value_compare`, 관계 연산자 | 공개 표면과 사전식 컨테이너 비교 | 필요한 마무리 기능이지만 기반 알고리즘 재사용 성격이 강하고 별도 면접 문제로는 밀도가 낮다. | 낮음 | 낮음 | 01·02 |
| C | 06 | `test(map): 삽입·검색·삭제 결과를 표준 map과 비교` | `tests/test_containers.cpp`; `test_map`, map 결과 비교 | 표준 컨테이너와의 공개 동작 차등 검증 | 기본 회귀 테스트로 중요하지만 구조 invariant·실패 경로를 직접 보지 않아 상세 문제로 만들지 않는다. | 중간 | 낮음 | 07 |
| B | 06 | `test(map): 역방향 순회와 경계 query 검증` | `tests/test_containers.cpp`; reverse traversal, `lower_bound`, `upper_bound` 경계 사례 | 종단 인접 경계와 역방향 순회 계약 | 좋은 경계 검증이지만 대표 구현 역량은 MAP-02와 SEN-03에 이미 포함된다. | 중간 | 낮음 | 08 |
| A · MAP-03 | 06 | `test(map): 범위 삭제 후 상태 검증` | `tests/test_containers.cpp`; range `erase` | 삭제 중 반복자 무효화와 다음 위치 저장 | 현재 노드를 지운 뒤 어떤 반복자를 계속 사용할지 판단하는 전형적인 안전 순회 문제다. | 높음 | 중간 | 07·08 |
| C | 06 | `test(map): 비교 함수 접근자 검증` | `tests/test_containers.cpp`; `key_comp`, `value_comp` | 상태를 가진 비교자의 공개 접근 | API 계약 확인 가치는 있지만 독립 구현 문제로 만들 만큼 복잡하지 않다. | 낮음 | 낮음 | 09 |

### Thread 07 — 레드-블랙 균형화와 구조·복잡도 invariant
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| S · RBT-01 | 07 | `feat(map): 레드-블랙 삽입 회전과 색 보정 구현` | `include/ft_map.hpp`; `_is_red`, `_is_black`, `_rotate_left`, `_rotate_right`, `_insert_fixup` | 레드-블랙 삽입 보정과 회전 invariant | 자료구조 면접의 대표 고난도 지점으로, 정렬·부모 링크·색·검정 높이를 동시에 보존해야 한다. | 높음 | 높음 | 06·08 |
| B | 07 | `test(map): 정렬 입력 삽입과 검색 경계 stress 검증` | `tests/test_containers.cpp`; 정렬·역정렬 입력, 검색 경계 stress | 편향 입력에 대한 균형 tree 회귀 | 퇴화 방지 의도는 중요하지만 구조·복잡도 검증은 뒤의 invariant와 계측 테스트가 더 대표적이다. | 중간 | 낮음 | 07 |
| S · RBT-02 | 07 | `feat(map): 레드-블랙 삭제 보정 구현` | `include/ft_map.hpp`; `_erase_node`, `_erase_fixup`, 회전·색 보정 | 검정 노드 삭제 뒤 black deficit 복구 | NULL을 검정 leaf로 취급하고 sibling 경우를 대칭 처리하는 가장 까다로운 상태 복구 경로다. | 높음 | 높음 | 06·08 |
| B | 07 | `test(map): 반복 삭제·복사·대입·교환 stress 검증` | `tests/test_containers.cpp`; 반복 key/iterator erase, copy, assignment, swap | 복합 연산 시퀀스의 공개 상태 회귀 | 실제 버그 탐지에는 유용하지만 개별 핵심 역량은 RBT-02, SEN-02, MTX-02에 분산된다. | 중간 | 낮음 | 08·09 |
| A · TEST-02 | 07 | `test(map): 무작위 연산마다 레드-블랙 불변식 검증` | `tests/support/map_inspector.hpp`, `tests/test_map_randomized.cpp`; `map_validation`, `map_inspector::validate`, `_validate_subtree`, `random_seed`, `operation_log` | 내부 invariant 검사와 재현 가능한 차등 상태 테스트 | 정렬 결과만 맞는 손상 tree를 잡고 실패 prefix를 재현하는 일반화 가능한 테스트 설계다. | 높음 | 중간 | 06·08 |
| A · TEST-03 | 07 | `perf(map): 높이와 비교 횟수 회귀 상한 추가` | `tests/test_complexity.cpp`; `counting_less`, `ceil_log2`, `check_scenario` | 점근 복잡도를 관측 가능한 회귀 조건으로 변환 | 기능 테스트가 통과해도 선형 퇴화할 수 있으므로 높이와 비교 호출 수를 별도 계측하는 가치가 높다. | 높음 | 중간 | 06 |

### Thread 08 — header sentinel과 map 반복자 안정성
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A · SEN-03 | 08 | `feat(map): 상수와 역방향 반복자 구현` | `include/ft_map.hpp`; `const_iterator`, `reverse_iterator`, `const_reverse_iterator` | const 반복자와 reverse adapter | Thread 06의 구현 지점을 반복하지만 최종 sentinel 반복자 계약을 이해하기 위한 직접 연관 위치다. | 높음 | 중간 | 01·06 |
| A · SEN-03 | 08 | `feat(map): 가변·상수 반복자 상호 비교 지원` | `include/ft_map.hpp`; `iterator`, `const_iterator`, 혼합 `operator==`, `operator!=` | 서로 다른 반복자 형식의 대칭 비교 | 한 방향 변환만 허용하면서 양쪽 피연산자 순서에서 같은 의미를 보장하는 템플릿 인터페이스 문제다. | 높음 | 중간 | 01·06 |
| B | 08 | `test(map): 가변·상수 반복자 비교 검증` | `tests/test_map_iterators.cpp`; `test_mixed_iterator_comparisons` | 혼합 반복자 비교의 대칭성 | SEN-03의 계약 검증으로 중요하지만 별도 상세 문제는 중복이다. | 중간 | 낮음 | 08 |
| B | 08 | `test(map): 역방향 순회와 경계 query 검증` | `tests/test_containers.cpp`; reverse traversal, 경계 query | 역방향 종단과 검색 경계의 결합 | GEN-02·MAP-02·SEN-03에서 이미 대표 문제로 다루므로 설명용 연관 사례로 남긴다. | 중간 | 낮음 | 01·06 |
| S · SEN-01 | 08 | `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | `include/ft_map.hpp`; `node_base`, `node`, `_header`, `_root`, `_value`, `_refresh_header`, `begin`, `end` | 값 없는 header sentinel과 안정된 end 표현 | end가 현재 root를 캡처하지 않고 구조 변경을 따라가며, key의 기본 생성 요구도 제거하는 핵심 표현 선택이다. | 높음 | 높음 | 06·07·09 |
| S · SEN-02 | 08 | `test(map): 회전·삭제·교환 뒤 반복자 상태 검증` | `tests/test_map_iterators.cpp`; `test_saved_end_after_rotation`, `test_saved_end_after_root_erasure`, `test_element_iterator_survives_rotation`, `test_iterators_across_swap`, `test_header_does_not_hold_a_value` | 구조 변경과 swap 뒤 반복자 안정성 | 노드 주소 안정성, 부모 링크 순회, 저장된 end, 새 소유 컨테이너의 종단을 한꺼번에 검증하는 고가치 경계다. | 높음 | 높음 | 07·09 |

### Thread 09 — map 소유권 트랜잭션과 정책 객체 예외 안전성
| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| A · MAP-01 | 09 | `feat(map): 노드 소유권과 빈 tree 상태 구현` | `include/ft_map.hpp`; `_alloc`, `_node_alloc`, `_create_node`, `_destroy_node`, `_clear` | 노드 단위 소유권과 소멸 순서 | Thread 06의 기반 구현을 예외 안전성 관점에서 다시 연결하는 대표 리소스 관리 지점이다. | 높음 | 중간 | 06 |
| A · MTX-01, MTX-02 | 09 | `feat(map): 삽입과 첨자 및 복사 동작 구현` | `include/ft_map.hpp`; `insert`, `operator[]`, range/copy constructor, `operator=` | tree 변경의 커밋 지점과 복사 소유권 | 이후 비교·할당 실패를 격리하는 트랜잭션 설계가 필요해진 출발점이다. | 높음 | 높음 | 06 |
| A · MAP-01 | 09 | `fix(map): 값 allocator 상태로 노드 allocator 구성` | `include/ft_map.hpp`; `_alloc`, `_node_alloc`, `node_allocator(alloc)` | allocator rebind 시 상태 보존 | 타입만 바꾸고 allocator 상태를 잃으면 잘못된 자원 공급자에게 해제할 수 있어 일반화 가치가 높다. | 높음 | 중간 | 06 |
| S · MTX-01 | 09 | `fix(map): 삽입 위치를 노드 할당 전에 확정` | `include/ft_map.hpp`; `insert`, `insert_left`, `_create_node` | 비교 단계와 노드 소유권 취득의 순서 | 예외 가능한 비교를 모두 끝낸 뒤 할당해야 연결 전 미소유 노드와 롤백 복잡도를 없앨 수 있다. | 높음 | 높음 | 06 |
| S · MTX-02 | 09 | `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리` | `include/ft_map.hpp`; range/copy constructor, `operator=`, `clear`, `_swap_tree_and_compare` | 생성자 정리와 copy-and-commit 강한 보장 | 부분 생성 객체의 소멸자 미호출 문제와 원본 대상 보존을 명시적 임시 소유권으로 해결한다. | 높음 | 높음 | 05·06 |
| A · TEST-01 | 09 | `test(map): 비교·할당 실패 시 노드 소유권 검증` | `tests/test_map_exceptions.cpp`; `test_insert_does_not_compare_after_allocation`, `test_range_constructor_rollback`, `test_copy_constructor_rollback`, `test_assignment_preserves_original` | 비교자·allocator 실패 주입과 노드 수명 계측 | 각 실패 지점에서 tree 보존과 outstanding block 수를 함께 확인하는 일반 실패 테스트 설계다. | 높음 | 중간 | 05 |
| S · SEN-01 | 09 | `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | `include/ft_map.hpp`; `node_base`, `node`, `_header`, `_refresh_header` | sentinel의 값 비소유 표현과 소유 tree 분리 | 반복자 안정성과 key 생성 제약을 동시에 해결하며 소유 노드와 비소유 경계 노드를 명확히 구분한다. | 높음 | 높음 | 08 |
| S · MTX-03 | 09 | `fix(map): 비교자 교환 실패 전에 tree 소유권 유지` | `include/ft_map.hpp`; `swap`, `_swap_tree_and_compare`, `_refresh_header` | 상태를 가진 정책 객체와 자원 소유권 교환 순서 | 비교자가 대입 전에 예외를 던지는 계약에서 정책 교환을 먼저 수행해 두 tree의 소유권이 섞이지 않게 한다. | 높음 | 높음 | 08 |
| A · TEST-01, MTX-03 | 09 | `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증` | `tests/test_map_policy_exceptions.cpp`; `throwing_compare`, `tracking_allocator`, `test_copy_assignment_keeps_target_ownership`, `test_public_swap_keeps_both_owners` | 정책 객체 예외 뒤 양쪽 컨테이너의 정렬·allocator·소유권 보존 | 정책과 tree를 함께 교환하는 연산의 실패 경계를 실제 상태와 자원 계측으로 검증한다. | 높음 | 중간 | 05·08 |

## 선별 결과 요약
- 전체 검토 커밋: **76개**
- S: **14개**, A: **26개**, B: **22개**, C: **14개**
- S/A 소스 행을 중복 역량별로 통합한 상세 면접 포인트: **23개**
- `stack`, README 시각화, 기본 공개 결과 비교, 빌드 설정 자체는 핵심 구현 문제 수를 늘리지 않고 B/C로 남겼다.

## 상세 워크북 파일 구성
| 파일 | 면접 포인트 | 역할 |
| --- | --- | --- |
| [01-cpp98-generic-and-iterators.md](01-cpp98-generic-and-iterators.md) | `GEN-01`, `GEN-02` | C++98 SFINAE, count/range 오버로드, reverse iterator의 one-past 표현 |
| [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md) | `VEC-01`, `VEC-02`, `VEC-03`, `VEC-04` | raw storage, 객체 수명, 재할당 트랜잭션, 용량 상한과 빈 반복자 |
| [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md) | `VEC-05`, `VEC-06`, `VEC-07` | 자기 별칭, assign/resize/push_back 롤백, 제자리·재할당 insert |
| [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md) | `MAP-01`, `MAP-02`, `MAP-03` | allocator rebind, 비교자 기반 lookup, BST 삭제와 이식 |
| [05-red-black-balancing.md](05-red-black-balancing.md) | `RBT-01`, `RBT-02` | 회전과 삽입 보정, 삭제 후 black deficit 복구 |
| [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md) | `SEN-01`, `SEN-02`, `SEN-03` | 값 없는 header, 저장된 end, 회전·삭제·swap 뒤 반복자 계약 |
| [07-map-ownership-transactions.md](07-map-ownership-transactions.md) | `MTX-01`, `MTX-02`, `MTX-03` | compare-before-allocate, 임시 tree 커밋, 정책 객체 교환 실패 |
| [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md) | `TEST-01`, `TEST-02`, `TEST-03` | 실패 주입, 구조 invariant 검사, 무작위 차등 검증, 복잡도 계측 |

## 대표 Thread와 연관 Thread 관계
| ID | 우선순위 | 대표 Thread / 커밋 | 연관 Thread | 상세 워크북 | 상세 주제 |
| --- | --- | --- | --- | --- | --- |
| `GEN-01` | A | Thread 01 / `feat(type-traits): CXX98 타입 선택 도구 구현` | 04·05 | [01-cpp98-generic-and-iterators.md](01-cpp98-generic-and-iterators.md#gen-01) | C++98에서 개수 오버로드와 범위 오버로드 분리 |
| `GEN-02` | A | Thread 01 / `feat(iterator): 역방향 반복자의 임의 접근 연산 완성` | 04·06·08 | [01-cpp98-generic-and-iterators.md](01-cpp98-generic-and-iterators.md#gen-02) | `reverse_iterator::base()`와 one-past 표현 |
| `VEC-01` | S | Thread 04 / `feat(vector): allocator 기반 저장소 수명 관리` | 05 | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-01) | 연속 저장소와 객체 수명 invariant |
| `VEC-02` | S | Thread 04 / `feat(vector): 용량 확장과 원소 재배치 구현` | 05 | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-02) | 재할당을 새 블록에서 완성한 뒤 커밋하기 |
| `VEC-03` | A | Thread 04 / `fix(vector): 용량 계산을 allocator 상한에서 포화` | 05 | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-03) | 용량 배수 성장의 overflow와 `max_size()` 경계 |
| `VEC-04` | A | Thread 04 / `fix(vector): allocator 형식과 빈 반복자 연산 보정` | 05 | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-04) | 빈 저장소에서 null 포인터 산술 피하기 |
| `VEC-05` | S | Thread 05 / `fix(vector): 자기 범위 assign과 insert 입력 보존` | 04 | [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md#vec-05) | 자기 참조 범위와 값 별칭을 snapshot으로 분리 |
| `VEC-06` | S | Thread 05 / `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리` | 04·09 | [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md#vec-06) | 생성·대입·크기 증가의 예외 보장 |
| `VEC-07` | S | Thread 05 / `fix(vector): fill·range 삽입의 객체 수명 보존` | 04 | [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md#vec-07) | 제자리 삽입에서 생성 영역과 대입 영역 분리 |
| `MAP-01` | A | Thread 09 / `fix(map): 값 allocator 상태로 노드 allocator 구성` | 06 | [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md#map-01) | allocator rebind와 노드 소유권 |
| `MAP-02` | S | Thread 06 / `feat(map): 검색과 경계 query 구현` | 07 | [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md#map-02) | 비교자 동치와 `find`·`lower_bound`·`upper_bound` |
| `MAP-03` | A | Thread 06 / `feat(map): 삭제와 clear 및 swap 구현` | 07·08 | [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md#map-03) | BST 삭제·`transplant`·범위 삭제 진행자 보존 |
| `RBT-01` | S | Thread 07 / `feat(map): 레드-블랙 삽입 회전과 색 보정 구현` | 06·08 | [05-red-black-balancing.md](05-red-black-balancing.md#rbt-01) | 회전과 삽입 보정으로 레드-블랙 invariant 복구 |
| `RBT-02` | S | Thread 07 / `feat(map): 레드-블랙 삭제 보정 구현` | 06·08 | [05-red-black-balancing.md](05-red-black-balancing.md#rbt-02) | 검정 노드 삭제 뒤 검정 높이 결손 복구 |
| `SEN-01` | S | Thread 08 / `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | 06·07·09 | [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md#sen-01) | 값을 갖지 않는 header sentinel과 `end()` 표현 |
| `SEN-02` | S | Thread 08 / `test(map): 회전·삭제·교환 뒤 반복자 상태 검증` | 07·09 | [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md#sen-02) | 노드 주소와 header를 이용한 반복자 안정성 |
| `SEN-03` | A | Thread 08 / `feat(map): 가변·상수 반복자 상호 비교 지원` | 01·06 | [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md#sen-03) | 가변·상수 반복자 변환과 대칭 비교 |
| `MTX-01` | S | Thread 09 / `fix(map): 삽입 위치를 노드 할당 전에 확정` | 06 | [07-map-ownership-transactions.md](07-map-ownership-transactions.md#mtx-01) | 비교 단계와 노드 소유권 획득을 분리한 삽입 트랜잭션 |
| `MTX-02` | S | Thread 09 / `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리` | 05·06 | [07-map-ownership-transactions.md](07-map-ownership-transactions.md#mtx-02) | 생성 rollback과 복사 대입의 임시 트리 commit |
| `MTX-03` | S | Thread 09 / `fix(map): 비교자 교환 실패 전에 tree 소유권 유지` | 08 | [07-map-ownership-transactions.md](07-map-ownership-transactions.md#mtx-03) | 정책 객체 예외와 트리 소유권 교환 순서 |
| `TEST-01` | A | Thread 09 / `test(map): 비교·할당 실패 시 노드 소유권 검증` | 05 | [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md#test-01) | 결정적 실패 주입과 객체·할당 자원 회계 |
| `TEST-02` | A | Thread 07 / `test(map): 무작위 연산마다 레드-블랙 불변식 검증` | 06·08 | [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md#test-02) | 구조 invariant 검사기와 재현 가능한 무작위 차등 테스트 |
| `TEST-03` | A | Thread 07 / `perf(map): 높이와 비교 횟수 회귀 상한 추가` | 06 | [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md#test-03) | 구조 높이와 비교자 호출 수로 복잡도 회귀 감지 |

## S/A 완전성 검증
아래 표는 마스터 선별표에서 S/A로 표시된 모든 행을 상세 문제 ID로 역추적한 결과다. 하나의 커밋이 여러 역량을 포함하면 둘 이상의 ID에 연결했고, 같은 역량의 반복 커밋은 대표 ID 하나에 통합했다.

| ID | 등급 | 통합된 S/A 근거 위치 | 상세 문서 | 상태 |
| --- | --- | --- | --- | --- |
| `GEN-01` | A | Thread 01 / `feat(type-traits): CXX98 타입 선택 도구 구현`<br>Thread 04 / `feat(vector): 크기 변경과 값 범위 할당 구현`<br>Thread 05 / `feat(vector): 크기 변경과 값 범위 할당 구현` | [01-cpp98-generic-and-iterators.md](01-cpp98-generic-and-iterators.md#gen-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `GEN-02` | A | Thread 01 / `feat(iterator): 역방향 반복자의 양방향 동작 구현`<br>Thread 01 / `feat(iterator): 역방향 반복자의 임의 접근 연산 완성` | [01-cpp98-generic-and-iterators.md](01-cpp98-generic-and-iterators.md#gen-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-01` | S | Thread 04 / `feat(vector): allocator 기반 저장소 수명 관리` | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-02` | S | Thread 04 / `feat(vector): 용량 확장과 원소 재배치 구현` | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-03` | A | Thread 04 / `fix(vector): 용량 계산을 allocator 상한에서 포화`<br>Thread 05 / `fix(vector): 용량 계산을 allocator 상한에서 포화` | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-03) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-04` | A | Thread 04 / `fix(vector): allocator 형식과 빈 반복자 연산 보정` | [02-vector-storage-and-capacity.md](02-vector-storage-and-capacity.md#vec-04) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-05` | S | Thread 05 / `fix(vector): 자기 범위 assign과 insert 입력 보존` | [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md#vec-05) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-06` | S | Thread 04 / `feat(vector): 크기 변경과 값 범위 할당 구현`<br>Thread 05 / `feat(vector): 크기 변경과 값 범위 할당 구현`<br>Thread 05 / `fix(vector): 저장소 교체와 크기 증가를 트랜잭션으로 처리` | [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md#vec-06) | 독립적인 상세 워크북 항목으로 작성됨 |
| `VEC-07` | S | Thread 04 / `feat(vector): 중간 변경 연산과 관계 비교 완성`<br>Thread 05 / `feat(vector): 중간 변경 연산과 관계 비교 완성`<br>Thread 05 / `fix(vector): fill·range 삽입의 객체 수명 보존`<br>Thread 05 / `test(vector): 삽입 복사·대입·할당 실패 sweep` | [03-vector-mutation-and-exception-safety.md](03-vector-mutation-and-exception-safety.md#vec-07) | 독립적인 상세 워크북 항목으로 작성됨 |
| `MAP-01` | A | Thread 06 / `feat(map): 노드 소유권과 빈 tree 상태 구현`<br>Thread 09 / `feat(map): 노드 소유권과 빈 tree 상태 구현`<br>Thread 09 / `fix(map): 값 allocator 상태로 노드 allocator 구성` | [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md#map-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `MAP-02` | S | Thread 06 / `feat(map): 검색과 경계 query 구현` | [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md#map-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `MAP-03` | A | Thread 06 / `feat(map): 삭제와 clear 및 swap 구현`<br>Thread 06 / `test(map): 범위 삭제 후 상태 검증` | [04-map-ordering-and-node-lifecycle.md](04-map-ordering-and-node-lifecycle.md#map-03) | 독립적인 상세 워크북 항목으로 작성됨 |
| `RBT-01` | S | Thread 07 / `feat(map): 레드-블랙 삽입 회전과 색 보정 구현` | [05-red-black-balancing.md](05-red-black-balancing.md#rbt-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `RBT-02` | S | Thread 07 / `feat(map): 레드-블랙 삭제 보정 구현` | [05-red-black-balancing.md](05-red-black-balancing.md#rbt-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `SEN-01` | S | Thread 08 / `fix(map): 값 없는 header로 끝 반복자 상태 안정화`<br>Thread 09 / `fix(map): 값 없는 header로 끝 반복자 상태 안정화` | [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md#sen-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `SEN-02` | S | Thread 08 / `test(map): 회전·삭제·교환 뒤 반복자 상태 검증` | [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md#sen-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `SEN-03` | A | Thread 06 / `feat(map): 상수와 역방향 반복자 구현`<br>Thread 08 / `feat(map): 상수와 역방향 반복자 구현`<br>Thread 08 / `feat(map): 가변·상수 반복자 상호 비교 지원` | [06-map-sentinel-and-iterator-stability.md](06-map-sentinel-and-iterator-stability.md#sen-03) | 독립적인 상세 워크북 항목으로 작성됨 |
| `MTX-01` | S | Thread 06 / `feat(map): 삽입과 첨자 및 복사 동작 구현`<br>Thread 09 / `feat(map): 삽입과 첨자 및 복사 동작 구현`<br>Thread 09 / `fix(map): 삽입 위치를 노드 할당 전에 확정` | [07-map-ownership-transactions.md](07-map-ownership-transactions.md#mtx-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `MTX-02` | S | Thread 06 / `feat(map): 삽입과 첨자 및 복사 동작 구현`<br>Thread 09 / `feat(map): 삽입과 첨자 및 복사 동작 구현`<br>Thread 09 / `fix(map): 생성과 복사 대입 실패를 임시 tree로 격리` | [07-map-ownership-transactions.md](07-map-ownership-transactions.md#mtx-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `MTX-03` | S | Thread 09 / `fix(map): 비교자 교환 실패 전에 tree 소유권 유지`<br>Thread 09 / `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증` | [07-map-ownership-transactions.md](07-map-ownership-transactions.md#mtx-03) | 독립적인 상세 워크북 항목으로 작성됨 |
| `TEST-01` | A | Thread 05 / `test(vector): 생성·대입·크기 변경 실패 주입`<br>Thread 05 / `test(vector): 삽입 복사·대입·할당 실패 sweep`<br>Thread 09 / `test(map): 비교·할당 실패 시 노드 소유권 검증`<br>Thread 09 / `test(map): 비교자 대입 실패 뒤 컨테이너 상태 검증` | [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md#test-01) | 독립적인 상세 워크북 항목으로 작성됨 |
| `TEST-02` | A | Thread 07 / `test(map): 무작위 연산마다 레드-블랙 불변식 검증` | [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md#test-02) | 독립적인 상세 워크북 항목으로 작성됨 |
| `TEST-03` | A | Thread 07 / `perf(map): 높이와 비교 횟수 회귀 상한 추가` | [08-failure-injection-invariants-and-complexity.md](08-failure-injection-invariants-and-complexity.md#test-03) | 독립적인 상세 워크북 항목으로 작성됨 |

- S/A 선별 행: **40개**
- 상세 ID가 없는 S/A 행: **0개**
- 상세 워크북 ID: **23개**
- 상세 문서가 없는 ID: **0개**

## 백지 구현 우선순위
1. **1차 — 수명·균형·소유권 핵심**: `VEC-01`, `VEC-02`, `VEC-07`, `RBT-01`, `RBT-02`, `SEN-01`, `MTX-01`, `MTX-02`
2. **2차 — 변경 경계와 검색·삭제**: `VEC-05`, `VEC-06`, `MAP-02`, `MAP-03`, `SEN-02`, `MTX-03`, `TEST-02`
3. **3차 — 기반 도구와 검증 계측**: `GEN-01`, `GEN-02`, `VEC-03`, `VEC-04`, `MAP-01`, `SEN-03`, `TEST-01`, `TEST-03`

## 설명 연습 우선순위
1. **1차**: raw storage와 객체 수명(`VEC-01`, `VEC-02`, `VEC-07`), 예외·소유권 트랜잭션(`VEC-06`, `MTX-01`, `MTX-02`, `MTX-03`, `TEST-01`), 레드-블랙 invariant(`RBT-01`, `RBT-02`, `TEST-02`), sentinel·반복자 안정성(`SEN-01`, `SEN-02`)
2. **2차**: 비교자 의미와 삭제(`MAP-02`, `MAP-03`), 용량 상한과 빈 포인터 경계(`VEC-03`, `VEC-04`), 자기 별칭(`VEC-05`), allocator rebind(`MAP-01`)
3. **3차**: C++98 오버로드 선택과 reverse iterator(`GEN-01`, `GEN-02`), const 반복자 상호 운용(`SEN-03`), 복잡도 회귀 계측(`TEST-03`)

## 한 문제로 통합한 Thread 묶음
- Thread 01의 type traits와 Thread 04·05의 범위 생성·`assign`·`insert` 제약 → `GEN-01`
- Thread 01의 reverse iterator와 Thread 04·06·08의 역방향 컨테이너 순회 → `GEN-02`
- Thread 04의 저장소·재할당 기반과 Thread 05의 실패 트랜잭션 → `VEC-01`, `VEC-02`, `VEC-06`
- Thread 04의 중간 변경 API와 Thread 05의 fill·range 삽입 수명·실패 경로 → `VEC-07`
- Thread 06의 노드 소유권과 Thread 09의 allocator 상태 보존 → `MAP-01`
- Thread 06의 삽입·복사 초기 구현과 Thread 09의 compare-before-allocate·임시 tree 커밋 → `MTX-01`, `MTX-02`
- Thread 06의 반복자 기반과 Thread 08·09의 값 없는 header·반복자 안정성 → `SEN-01`, `SEN-02`, `SEN-03`
- Thread 05의 `vector` 실패 주입과 Thread 09의 `map`·정책 객체 실패 주입 → `TEST-01`
- Thread 07의 구조 검사기·고정 seed 차등 테스트·반복 root 삭제 → `TEST-02`
