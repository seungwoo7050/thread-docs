# miniRT 개발자 기술면접 워크북 — 마스터 인덱스

## 문서 목적과 검토 범위

이 인덱스는 현재 GPT 프로젝트에서 실제로 확인할 수 있는 miniRT 개발 Thread 01~13의 문서와 커밋 기록만을 기준으로 작성했다. 원격 저장소 상태, 현재 브랜치의 최종 코드, 프로젝트 메모리에 나타나지 않은 파일과 식별자는 사용하지 않았다.

- 커밋 제목은 현재 프로젝트 기록에서 제목까지 확인된 경우에만 적었다.
- 같은 역량이 여러 Thread에서 반복되면 대표 면접 포인트 하나로 통합하고, 다른 위치는 `연관 Thread`와 `통합 상태`에 남겼다.
- 08-03에서 확인되는 도형 필드 비공개화, `Scene::shapeCount`, `Scene::shapeAt`, mutation boundary 검증은 구현 위치와 내용은 확인되지만 이 인덱스 작성 시 커밋 제목이 노출되지 않아 별도의 커밋명을 만들지 않았다. 관련 내용은 P06과 P11의 원본 확인 위치에만 반영했다.
- 우선순위는 면접에서의 질문 가치와 10~30분 백지 구현 가치에 따라 정했다. 프로젝트에 특수한 상수값, 반복 fixture, 빌드 배선만으로는 별도 문제를 만들지 않았다.

## 우선순위 기준

| 우선순위 | 의미 |
|---|---|
| S | 반드시 준비. 핵심 원리 설명과 직접 구현 모두 가치가 높다. |
| A | 준비 가치가 높다. 질문 또는 축소 구현 문제로 출제될 가능성이 높다. |
| B | 구현보다 설계 의도, 검증 전략, trade-off 설명을 준비하는 편이 낫다. |
| C | 별도 면접 문제로 만들 필요가 낮다. 다른 항목의 배경으로만 확인한다. |

## 전체 Thread·커밋 선별 결과

`질문형`과 `구현형`은 해당 기록을 독립적인 면접 문제로 사용할 때의 가치다. `통합`은 별도 문제를 만들지 않고 대표 포인트에 흡수했음을 뜻한다.

| 우선순위 | Thread | 커밋 메시지 | 관련 위치 | 핵심 면접 주제 | 선별 이유 | 질문형 | 구현형 | 연관 Thread | 상세 워크북 |
|---|---:|---|---|---|---|---|---|---|---|
| B | 01 | `feat(math): 벡터 값과 산술 연산 구현`<br>`feat(math): 벡터 길이와 기하 연산 구현` | `include/ray/math.hpp`, `src/math.cpp`, `Vec3`, `dot`, `cross`, `normalize` | 값 타입, 연산자, 벡터 기초 | 기반 지식은 중요하지만 단독 문제로는 구현이 평이하다. 수치 안정성 문제의 전제로 통합한다. | 중 | 중 | 02, 04, 07 | 통합 → [P01](01-numerics-and-intersections.md#p01) |
| C | 01 | `feat(math): 벡터 비교와 색상 범위 연산 추가` | `include/ray/math.hpp`, `src/math.cpp`, `Color`, 색상 범위 연산 | 보조 연산과 표현 편의 | 반복성이 높고 독립 면접 지점으로 얻는 정보가 적다. | 낮 | 낮 | 05, 11 | 없음 |
| S | 01 | `fix(math): 큰 유한 벡터를 안정적으로 정규화` | `src/math.cpp`, `Vec3::length`, `normalize` | 부동소수점 중간 overflow와 안정적 norm | 유한 입력이 중간 제곱에서 무한대로 바뀌는 실전 수치 오류를 다룬다. 언어·런타임의 수 표현과 방어 계약을 함께 질문할 수 있다. | 높 | 높 | 03, 04, 07 | [P01](01-numerics-and-intersections.md#p01) |
| B | 01 | `test(math): 큰 유한 벡터 정규화 검증` | `tests/core_tests.cpp` | 수치 회귀 입력 선정 | 핵심 구현은 P01이고, 이 커밋은 경계 입력을 고르는 법을 보강한다. | 중 | 낮 | 13 | 통합 → [P01](01-numerics-and-intersections.md#p01) |
| S | 02 | `feat(geometry): hit와 도형 교차 계약 정의`<br>`feat(geometry): 구 교차 계산 구현` | `include/ray/geometry.hpp`, `src/geometry.cpp`, `HitRecord`, `HitRecord::setFaceNormal`, `Shape::intersect`, `Sphere::intersect` | hit 불변식, 법선 방향, 유효한 t 구간, 근 선택 | 이후 모든 도형·BVH·셰이딩이 의존하는 핵심 계약이다. 내부 시작 광선, 접선, 뒤쪽 root를 함께 검증할 수 있다. | 높 | 높 | 05, 08 | [P02](01-numerics-and-intersections.md#p02) |
| A | 02 | `feat(geometry): 평면 교차 계산 구현` | `src/geometry.cpp`, `Plane::intersect` | 평행 판정과 구간 교차 | 분모가 0에 가까운 경우와 양방향 법선 처리를 설명하기 좋다. 구현은 P02에 통합한다. | 높 | 중 | 05, 07 | 통합 → [P02](01-numerics-and-intersections.md#p02) |
| S | 02 | `feat(geometry): 유한 원기둥 옆면 교차 구현`<br>`feat(geometry): 원기둥 cap과 최근접 hit 선택 완성` | `src/geometry.cpp`, `Cylinder::intersect`, `update_hit_if_closer`, `test_cylinder_cap` | 축 성분 분해, 유한 구간, side·cap 후보 통합 | 분석적 교차 중 가장 복합적이며, 후보 생성과 최근접 선택을 분리하는 설계가 일반화된다. | 높 | 높 | 03, 07 | [P03](01-numerics-and-intersections.md#p03) |
| B | 02 | `test(core): 수학·기하·파서·출력 회귀 기준 추가` | `tests/core_tests.cpp`, `CMakeLists.txt` | 작은 회귀 suite 구성 | 여러 기능을 얕게 묶는 smoke 성격이라 독립 구현보다 각 대표 문제의 자가 검증에 통합하는 편이 낫다. | 중 | 낮 | 01, 03, 11, 13 | 통합 → P01, P02, P03, P15 |
| S | 03 | `feat(parser): 유한 수와 범위 값 해석 구현`<br>`feat(parser): 벡터와 색상 token 해석 구현` | `include/ray/parser.hpp`, `src/parser.cpp`, `ParseError`, `parseDoubleToken`, `parseIntToken`, `parseRatio`, `parsePositiveDouble`, `parseVec3`, `parseColor` | 완전 소비, finite 검증, 범위·형식 계약, 진단 위치 | 외부 입력을 내부 불변식으로 바꾸는 경계다. `nan`, `inf`, trailing 문자, overflow, 토큰 개수 오류를 한 문제에서 다룰 수 있다. | 높 | 높 | 01, 12 | [P05](02-parser-scene-and-shading.md#p05) |
| A | 03 | `feat(parser): 카메라와 광원 지시어 지원`<br>`feat(parser): 원기둥 지시어 지원`<br>`feat(parser): 필수 지시어 검증과 입력 loader 완성` | `src/parser.cpp`, `parseScene`, `parseSceneText`, `parseSceneFile`, `loadScene`, `expectCount`, `rejectDuplicate`, `requireNonzeroVector` | 문법 상태, 중복·필수 지시어, 객체 구성 순서 | 개별 directive 코드는 반복적이지만, 중복 금지와 필수 상태 검증, 성공 후 가속 구조 구축 순서는 설계 질문 가치가 높다. | 높 | 중 | 04, 08 | 통합 → [P05](02-parser-scene-and-shading.md#p05), [P06](02-parser-scene-and-shading.md#p06) |
| S | 03 | `fix(parser): 임계값 이하 방향 벡터 거부` | `src/parser.cpp`, `requireNonzeroVector`, `kEpsilon` | 표현별 near-zero 판정 차이 | 성분별 판정과 벡터 길이 판정이 다른 입력을 만든다는 점이 좋은 경계 조건이다. | 높 | 중 | 01, 04 | 통합 → [P05](02-parser-scene-and-shading.md#p05) |
| B | 03 | `test(parser): 퇴화한 카메라와 원기둥 방향 검증` | `tests/core_tests.cpp` | 퇴화 입력 회귀 | 핵심은 P05의 입력 계약이며, 테스트 커밋은 반례 선정 근거로 사용한다. | 중 | 낮 | 01, 04 | 통합 → [P05](02-parser-scene-and-shading.md#p05) |
| S | 03 | `feat(scene): 가속 구조 소유권과 rebuild 경계 구성` | `include/ray/scene.hpp`, `src/scene.cpp`, `Scene::addShape`, `Scene::buildAcceleration`, `Scene::accelerationReady`, `Scene::intersect` | 파생 인덱스의 lifecycle, stale 상태 방지, fallback | 원본 도형 집합과 파생 BVH의 정합성을 상태 전이로 설명할 수 있다. 변경 후 invalidation 누락은 실제 correctness 버그가 된다. | 높 | 높 | 08 | [P06](02-parser-scene-and-shading.md#p06) |
| A | 03 | `feat(parser): 선택적 도형 재질 문법 추가` | `src/parser.cpp`, 선택적 `diffuse`·`metal` token | 하위 호환 문법 확장과 기본값 | 선택 인자, 허용 집합, 누락 기본값, 미지 값 거부를 API 계약 관점에서 묻기 좋다. 핵심 재질 dispatch에 통합한다. | 높 | 중 | 06, 12 | 통합 → [P08](02-parser-scene-and-shading.md#p08) |
| B | 03 | `test(material): 재질 파싱과 반사 깊이 검증` | `tests/material_tests.cpp` | 새 기능과 기존 diffuse 결과의 동시 회귀 | 구현보다 회귀 보호 전략을 설명하는 항목이다. | 중 | 낮 | 06, 13 | 통합 → [P08](02-parser-scene-and-shading.md#p08) |
| A | 04 | `feat(camera): 화면 좌표를 카메라 광선으로 변환`<br>`perf(camera): 픽셀별 카메라 프레임 재계산 제거` | `include/ray/camera.hpp`, `src/camera.cpp`, `CameraFrame`, `buildCameraFrame`, `makeCameraRay`, `src/renderer.cpp` | 직교 카메라 기저, FOV 투영, 프레임 캐시 | 벡터 기본기와 좌표계 선택, 퇴화한 up 벡터, 계산 재사용의 trade-off를 함께 확인할 수 있다. | 높 | 높 | 01, 09 | [P04](01-numerics-and-intersections.md#p04) |
| B | 04 | `test(camera): 재사용한 카메라 프레임의 동치 검증` | `tests/core_tests.cpp` | 최적화 전후 동치 | 최적화가 의미를 바꾸지 않았음을 보이는 좋은 테스트지만 독립 문제는 아니다. | 중 | 낮 | 13 | 통합 → [P04](01-numerics-and-intersections.md#p04) |
| B | 05 | `feat(material): diffuse 재질 값 모델 추가`<br>`feat(scene): 카메라·조명과 장면 aggregate 구성` | `include/ray/material.hpp`, `include/ray/scene.hpp`, `Material`, `Light`, `Camera`, `Scene` | 도메인 값 모델과 aggregate 경계 | 값 객체 자체는 단순하다. 소유권과 셰이딩 문제의 배경으로 충분하다. | 중 | 낮 | 03, 06, 08 | 통합 → P06, P07, P08 |
| A | 05 | `feat(scene): 선형 최근접 교차 탐색 구현` | `src/scene.cpp`, `Scene::intersect`, `findNearestHit` | 현재 최단 거리로 탐색 구간 축소 | O(n) 기준 구현이 BVH의 correctness oracle이 되며 equal-distance 정책까지 연결된다. | 높 | 중 | 02, 08 | 통합 → [P02](01-numerics-and-intersections.md#p02), [P11](03-acceleration-and-performance.md#p11) |
| A | 05 | `feat(render): 직접광과 그림자 추적 구현` | `include/ray/renderer.hpp`, `src/shading.cpp`, `isOccluded`, `shadeHit`, `traceRay` | Lambert 조명, shadow segment, self-intersection bias | 무한 광선이 아니라 광원까지의 유한 구간을 검사해야 하며, bias가 만드는 정확도 trade-off가 뚜렷하다. | 높 | 높 | 01, 02, 06 | [P07](02-parser-scene-and-shading.md#p07) |
| B | 05 | `test(render): 장면 렌더링 smoke 검사 추가`<br>`test(output): PPM과 렌더링 체크섬 기준 고정` | `tests/render_smoke.sh`, `tests/core_tests.cpp` | end-to-end smoke와 golden checksum | 변경 감지는 유용하지만 checksum만으로 원인을 설명하기 어렵다. 결정성 검증 항목에 통합한다. | 중 | 낮 | 09, 11, 13 | 통합 → P12, P13, P15 |
| A | 06 | `feat(material): metal 모델과 깊이 제한 반사 구현` | `include/ray/material.hpp`, `src/scene.cpp`, `src/shading.cpp`, `MaterialType`, `traceRay` | material dispatch, 재귀 종료, 반사 광선 bias | 재귀 깊이의 의미, 종료값, secondary ray 계측, diffuse 경로 보존을 질문하기 좋다. | 높 | 높 | 03, 05, 09 | [P08](02-parser-scene-and-shading.md#p08) |
| A | 06 | `feat(cli): 반사 깊이 option과 기본값 추가` | `src/main.cpp`, `src/renderer.cpp`, `RenderSettings::maxDepth` | 내부 정책을 외부 계약으로 노출 | 범위 0..32, 기본값, 중복 옵션 거부가 CLI 파서 문제와 직접 연결된다. | 높 | 중 | 12 | 통합 → [P17](05-output-and-cli-contracts.md#p17) |
| B | 07 | `feat(accel): AABB 값과 결합 연산 구현` | `include/ray/accel.hpp`, `src/accel.cpp`, `Aabb`, `surroundingBox` | 유효한 box와 결합 | 자체 구현은 짧다. slab 교차와 BVH 구축의 전제로 통합한다. | 중 | 중 | 08 | 통합 → P09, P11 |
| S | 07 | `feat(accel): ray-box slab 교차 구현` | `Aabb::intersect` | 구간 축소, 평행 축, 음수 방향, 경계 포함 | BVH pruning의 정확도를 결정하는 짧고 밀도 높은 알고리즘 문제다. | 높 | 높 | 08 | [P09](03-acceleration-and-performance.md#p09) |
| A | 07 | `feat(accel): 도형 경계 계약과 구·평면 bounds 추가`<br>`feat(accel): 원기둥의 보수적 bounds 계산 추가` | `Shape::bounds`, `Sphere::bounds`, `Plane::bounds`, `Cylinder::bounds` | bounded/unbounded 분리와 보수적 경계 | 빠른 구조보다 false negative가 없는 경계가 우선이다. 임의 축 원기둥과 outward rounding이 좋은 trade-off 질문이다. | 높 | 높 | 02, 08 | [P10](03-acceleration-and-performance.md#p10) |
| B | 07 | `test(accel): AABB와 도형 경계 계산 검증` | `tests/core_tests.cpp` | 방향·경계·보수성 반례 | 대표 구현 P09와 P10의 자가 검증으로 통합한다. | 중 | 낮 | 13 | 통합 → P09, P10 |
| S | 08 | `refactor(scene): 장면 도형의 단독 소유권 적용`<br>`feat(scene): 가속 구조 소유권과 rebuild 경계 구성` | `Scene`, `std::unique_ptr<Shape>`, `Scene::addShape`, `Scene::buildAcceleration`, private shape storage와 조회 경계 | 소유권, mutation boundary, 파생 데이터 무효화 | shared ownership이 필요 없는 객체를 단독 소유하고, 외부 변경으로 bounds와 BVH가 어긋나지 않게 막는 설계다. | 높 | 높 | 03 | [P06](02-parser-scene-and-shading.md#p06) |
| B | 08 | `feat(accel): BVH node와 연속 저장소 구성` | `BvhPrimitive`, `BvhNode`, `Bvh`, 연속 `nodes_`·`primitiveIndices_` | 트리의 배열 표현과 locality | 중요한 설계 선택이지만 단독 구현보다는 전체 BVH 문제의 일부로 묻는 편이 낫다. | 높 | 중 | 07, 10 | 통합 → [P11](03-acceleration-and-performance.md#p11) |
| S | 08 | `feat(accel): 결정적 중앙 분할 BVH 구축 구현`<br>`feat(accel): 선형·BVH 탐색 모드 계약 연결`<br>`feat(accel): 결정적 BVH 최근접 순회 구현` | `Bvh::build`, `Bvh::buildNode`, `Scene::intersect`, `AccelMode`, `BvhNode::isLeaf` | 결정적 build, near-first traversal, 최근접·tie 동치 | 성능 구조가 기준 구현의 결과를 바꾸지 않아야 한다. stable ordering, leaf 기준, unbounded 처리, equal-distance 정책까지 한 문제로 묶인다. | 높 | 높 | 02, 07, 10 | [P11](03-acceleration-and-performance.md#p11) |
| A | 08 | 선형·BVH 동치 및 dense scene 회귀 기록 | `tests/accel_tests.cpp`, `requireEquivalentHit`, equal-distance·empty·plane-only·cylinder·dense render 검사 | 최적화의 differential oracle | 구현 경로가 다른 두 모드를 비교하면 복잡한 expected 값을 직접 만들지 않고 correctness를 검증할 수 있다. | 높 | 중 | 09, 10, 13 | 통합 → P11, P12, P13 |
| A | 09 | `perf(render): 광선과 교차 작업량 계측 추가` | `RenderStats`, `renderScene`, `Scene::intersect`, `shadeHit`, `traceRay` | 관측 지점과 작업량 의미 | wall-clock만으로 설명하기 어려운 성능을 알고리즘 작업량으로 분해한다. 상세 문제는 재현 가능한 벤치마크로 통합한다. | 높 | 중 | 10 | 통합 → [P12](03-acceleration-and-performance.md#p12) |
| S | 09 | `feat(renderer): 작업자 수 설정과 자동 선택 추가` 및 Thread 09-01의 고정 tile 병렬 렌더링 변경 | `include/ray/renderer.hpp`, `src/renderer.cpp`, `RenderSettings::threadCount`, `renderScene`, 고정 tile, `next_tile`, per-worker `RenderStats` | 비중첩 쓰기, 동적 작업 배분, 결정적 결과, 통계 reduction | 스케줄은 비결정적이어도 각 픽셀의 소유와 계산은 결정적이어야 한다. data race·false sharing·과도한 worker 수를 함께 묻기 좋다. | 높 | 높 | 10, 13 | [P13](04-concurrency-and-determinism.md#p13) |
| A | 09 | `test(render): 작업자 수에 따른 함수 결과 동치 검증`<br>`test(render): 실행 모드별 PPM byte 결정성 검증` | `tests/render_tests.cpp`, `tests/render_determinism.sh` | 함수 결과·통계·파일 byte의 다층 결정성 | 이미지 equality만이 아니라 checksum, 작업량, 최종 PPM byte까지 비교하는 검증 설계가 좋다. | 높 | 중 | 08, 10, 13 | 통합 → [P13](04-concurrency-and-determinism.md#p13) |
| S | 09 | `fix(renderer): 작업자 예외를 호출자에게 전달` | `src/renderer.cpp`, `std::exception_ptr`, stop 신호, `ThreadJoiner` | thread 경계 예외, cooperative stop, join 보장 | worker 예외를 놓치면 terminate·부분 성공·resource leak이 생긴다. 실패 전파 순서와 RAII를 직접 구현할 가치가 높다. | 높 | 높 | 11, 13 | [P14](04-concurrency-and-determinism.md#p14) |
| B | 09 | `test(renderer): 작업자 실패 전파와 회수 검증` | `tests/render_tests.cpp`, `ThrowingShape`, `testWorkerExceptionPropagation` | 의도적 worker 실패 주입 | 구현 문제 P14의 실패 경로 자가 검증으로 통합한다. | 높 | 낮 | 13 | 통합 → [P14](04-concurrency-and-determinism.md#p14) |
| A | 10 | `perf(benchmark): 조밀 장면 기준 workload 추가`<br>`perf(benchmark): 반복 측정과 결정성 보고 구성`<br>`perf(benchmark): 선형 탐색과 BVH 작업량 비교`<br>`perf(benchmark): 측정 schema와 가속 기준 검증 고정` | `benchmarks/render_benchmark.cpp`, `RenderStats`, `checksumHex`, `AccelMode` | warm-up, 반복·median, 결과 동치, workload 기준 | 성능 수치를 신뢰할 수 있게 만드는 실험 설계다. 시간과 결정적 작업량을 분리하고 성능 gate의 취약점도 설명할 수 있다. | 높 | 중 | 08, 09, 13 | [P12](03-acceleration-and-performance.md#p12) |
| C | 10 | `perf(benchmark): 참조 측정값 기록` | `benchmarks/reference.json` | 특정 환경의 기준 수치 | 환경 종속 숫자 자체는 암기할 가치가 없다. schema와 재현 조건만 P12에서 다룬다. | 낮 | 낮 | 13 | 통합 → [P12](03-acceleration-and-performance.md#p12) |
| B | 11 | `feat(output): PPM 직렬화와 이미지 체크섬 구현`<br>`fix(output): 표준 FNV-1a 기준값 적용`<br>`test(output): PPM과 렌더링 체크섬 기준 고정` | `include/ray/output.hpp`, `src/output.cpp`, `writePpm`, `checksumHex` | 직렬화 경계와 회귀 fingerprint | 포맷과 checksum 자체보다 입력 invariant와 실패 시 보존이 더 높은 면접 가치다. | 중 | 중 | 05, 09 | 통합 → P15, P16 |
| S | 11 | `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지`<br>`fix(output): 불일치한 이미지 저장소 거부` | `include/ray/renderer.hpp`, `src/renderer.cpp`, `src/output.cpp`, `Image`, `pixelStorageSize`, `Image::validate` | 곱셈 overflow, signed→unsigned 변환, 저장소 invariant | 할당 전 검증과 사용 전 검증이 모두 필요하다. 잘못된 크기가 OOB와 기존 파일 손상으로 이어지는 경계를 다룬다. | 높 | 높 | 09, 12 | [P15](05-output-and-cli-contracts.md#p15) |
| B | 11 | `test(image): 잘못된 차원과 저장 크기 계산 검증`<br>`test(output): 잘못된 이미지 저장소 처리 검증` | `tests/core_tests.cpp` | overflow·short/excess storage 반례 | P15의 자가 검증에 통합한다. | 중 | 낮 | 13 | 통합 → [P15](05-output-and-cli-contracts.md#p15) |
| S | 11 | `fix(output): PPM 출력 실패 시 기존 파일 보존` | `src/output.cpp`, `writePpm(Image, ostream)`, `writePpm(Image, path)`, `TemporaryOutput`, `replaceFile` | 임시 파일, atomic replacement, commit/rollback, cleanup | 파일 쓰기는 중간 실패가 정상 경로만큼 중요하다. 기존 대상 보존과 임시 파일 회수라는 트랜잭션 성질을 확인할 수 있다. | 높 | 높 | 12, 13 | [P16](05-output-and-cli-contracts.md#p16) |
| B | 11 | `test(output): 출력 실패의 대상 보존과 정리 검증` | `tests/output_tests.cpp`, `FailingBuffer`, `TestDirectory` | fault injection과 resource cleanup 검증 | P16의 실패 경로 설계를 검증하는 대표 테스트다. | 높 | 낮 | 13 | 통합 → [P16](05-output-and-cli-contracts.md#p16) |
| C | 12 | `chore(project): CXX17 실행 골격과 직접 빌드 구성` | `Makefile`, `src/main.cpp` | 빌드 골격과 usage 출력 | 프로젝트 시작 boilerplate라 별도 면접 항목으로 만들 필요가 낮다. | 낮 | 낮 | 13 | 없음 |
| B | 12 | `feat(cli): 장면 렌더링 명령 연결` | `src/main.cpp`, `main` | application orchestration과 종료 경계 | parse→render→persist 순서와 exception→exit code 변환은 설명 가치가 있으나 구현은 단순하다. | 중 | 낮 | 03, 11 | 통합 → [P17](05-output-and-cli-contracts.md#p17) |
| A | 12 | `feat(cli): 가속 방식 선택 option 추가`<br>`feat(cli): 작업자 수 option 추가`<br>`feat(cli): 반사 깊이 option과 기본값 추가` | `src/main.cpp`, `CliOptions`, `parseUnsigned`, `parseCli`, `printUsage` | strict option parser, 중복·누락·범위·overflow, 기본값 | 작은 상태 기계이면서 입력 보안과 API 계약을 확인할 수 있다. 라이브러리 의존 없이 20분 내 구현하기 좋다. | 높 | 높 | 06, 09 | [P17](05-output-and-cli-contracts.md#p17) |
| A | 12 | `test(cli): 렌더링 옵션과 오류 종료 계약 검증` | `tests/cli_contract.sh` | usage error 2, runtime error 1, success 0, 경계 옵션 | 옵션 parser의 정상·실패 경로를 프로세스 수준에서 검증한다. P17에 통합한다. | 높 | 중 | 13 | 통합 → [P17](05-output-and-cli-contracts.md#p17) |
| A | 13 | `test(render): smoke 검사의 fixture와 실행 경로 정리`<br>`test(render): 실행 모드별 PPM byte 결정성 검증`<br>`test(output): 출력 실패의 대상 보존과 정리 검증` | `tests/render_smoke.sh`, `tests/render_determinism.sh`, `tests/output_tests.cpp` | 기능 gate의 oracle과 실패 artifact | 새로운 알고리즘 문제가 아니라 P13·P16·P17의 검증 레이어다. 대표 문제에 명시적으로 통합한다. | 높 | 중 | 09, 11, 12 | 통합 → P13, P16, P17 |
| B | 13 | `build: expose deterministic verification targets` | `CMakeLists.txt`, `Makefile`, CTest timeout, `check`, `ci`, `sanitize` | 재현 가능한 검증 DAG와 timeout | 시스템 경계와 자동화 설계 질문 가치는 높지만 miniRT 핵심 구현보다 우선순위가 낮다. | 높 | 낮 | 10, 12 | 설명 연습만 |
| B | 13 | cross-platform CI와 sanitizer gate 변경 기록 | `.github/workflows/cpp-miniRT-ci.yml`, Ubuntu/macOS compiler matrix, ASan·UBSan, 실패 로그 artifact | 환경 다양성, sanitizer, 최소 권한, 실패 진단 | 특정 YAML 암기보다 왜 독립 job·고정 toolchain·진단 artifact가 필요한지 설명하는 항목이다. | 높 | 낮 | 10, 11 | 설명 연습만 |
| B | 13 | 안전한 build directory 정리 경계 | `Makefile`, `guard-build-dir`, `clean` | destructive command의 경로 검증 | 빌드 설정 중 드물게 보안·운영 사고와 직접 연결되는 지점이다. 짧은 설계 질문으로 준비한다. | 중 | 낮 | 11, 12 | 설명 연습만 |

## 대표 면접 포인트와 상세 문서

| ID | 우선순위 | 대표 Thread·커밋 | 면접 포인트 | 연관 Thread | 상세 문서 | 작성 상태 |
|---|---|---|---|---|---|---|
| P01 | S | 01 / `fix(math): 큰 유한 벡터를 안정적으로 정규화` | 안정적인 벡터 norm과 near-zero 계약 | 03, 04, 07 | [01-numerics-and-intersections.md](01-numerics-and-intersections.md#p01) | 독립 항목 |
| P02 | S | 02 / `feat(geometry): hit와 도형 교차 계약 정의` | HitRecord, 법선 방향, t interval, 구·평면 최근접 교차 | 05, 08 | [01-numerics-and-intersections.md](01-numerics-and-intersections.md#p02) | 독립 항목; Thread 05 선형 탐색 통합 |
| P03 | S | 02 / `feat(geometry): 원기둥 cap과 최근접 hit 선택 완성` | 유한 원기둥 side·cap 후보 통합 | 03, 07 | [01-numerics-and-intersections.md](01-numerics-and-intersections.md#p03) | 독립 항목 |
| P04 | A | 04 / `feat(camera): 화면 좌표를 카메라 광선으로 변환` | 카메라 직교 프레임과 픽셀 광선, 프레임 재사용 | 01, 09 | [01-numerics-and-intersections.md](01-numerics-and-intersections.md#p04) | 독립 항목 |
| P05 | S | 03 / `feat(parser): 유한 수와 범위 값 해석 구현` | strict parser, finite·범위·진단 계약 | 01, 12 | [02-parser-scene-and-shading.md](02-parser-scene-and-shading.md#p05) | 독립 항목; directive 반복 구현 통합 |
| P06 | S | 03·08 / `feat(scene): 가속 구조 소유권과 rebuild 경계 구성` | Scene 단독 소유권과 BVH invalidation·fallback·rebuild | 03, 08 | [02-parser-scene-and-shading.md](02-parser-scene-and-shading.md#p06) | 독립 항목; 08-03 mutation boundary 통합 |
| P07 | A | 05 / `feat(render): 직접광과 그림자 추적 구현` | Lambert 직접광, shadow segment, ray bias | 01, 02, 06 | [02-parser-scene-and-shading.md](02-parser-scene-and-shading.md#p07) | 독립 항목 |
| P08 | A | 06 / `feat(material): metal 모델과 깊이 제한 반사 구현` | 재질 dispatch와 깊이 제한 반사 | 03, 05, 12 | [02-parser-scene-and-shading.md](02-parser-scene-and-shading.md#p08) | 독립 항목; 선택적 재질 문법 통합 |
| P09 | S | 07 / `feat(accel): ray-box slab 교차 구현` | AABB slab 교차 | 08 | [03-acceleration-and-performance.md](03-acceleration-and-performance.md#p09) | 독립 항목 |
| P10 | A | 07 / `feat(accel): 원기둥의 보수적 bounds 계산 추가` | bounded/unbounded와 보수적 도형 경계 | 02, 08 | [03-acceleration-and-performance.md](03-acceleration-and-performance.md#p10) | 독립 항목 |
| P11 | S | 08 / `feat(accel): 결정적 중앙 분할 BVH 구축 구현` | 결정적 BVH build·traversal과 선형 동치 | 02, 07, 10 | [03-acceleration-and-performance.md](03-acceleration-and-performance.md#p11) | 독립 항목; equal-distance·differential test 통합 |
| P12 | A | 10 / `perf(benchmark): 측정 schema와 가속 기준 검증 고정` | 작업량 계측과 재현 가능한 benchmark evidence | 08, 09, 13 | [03-acceleration-and-performance.md](03-acceleration-and-performance.md#p12) | 독립 항목; Thread 09 계측·Thread 13 gate 통합 |
| P13 | S | 09 / `feat(renderer): 작업자 수 설정과 자동 선택 추가` | 결정적 tiled parallel rendering과 per-worker reduction | 10, 13 | [04-concurrency-and-determinism.md](04-concurrency-and-determinism.md#p13) | 독립 항목; 결과·checksum·PPM byte 동치 통합 |
| P14 | S | 09 / `fix(renderer): 작업자 예외를 호출자에게 전달` | worker 실패의 stop·join·rethrow | 11, 13 | [04-concurrency-and-determinism.md](04-concurrency-and-determinism.md#p14) | 독립 항목 |
| P15 | S | 11 / `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지` | 이미지 크기 계산과 저장소 invariant | 09, 12 | [05-output-and-cli-contracts.md](05-output-and-cli-contracts.md#p15) | 독립 항목; 불일치 저장소 검증 통합 |
| P16 | S | 11 / `fix(output): PPM 출력 실패 시 기존 파일 보존` | 실패 안전한 직렬화와 atomic replacement | 12, 13 | [05-output-and-cli-contracts.md](05-output-and-cli-contracts.md#p16) | 독립 항목; fault injection test 통합 |
| P17 | A | 12 / `test(cli): 렌더링 옵션과 오류 종료 계약 검증` | strict CLI option parser와 exit-code contract | 06, 09, 13 | [05-output-and-cli-contracts.md](05-output-and-cli-contracts.md#p17) | 독립 항목; 각 옵션 추가 커밋 통합 |

## S/A 완전성 대조

아래 상태표는 상세 문서 작성 후의 최종 대조 결과다. `통합 대상`은 별도 문제로 남기지 않고 해당 대표 항목의 질문, 자가 검증 또는 원본 확인 위치에 명시한 기록이다.

| ID | 우선순위 | 상태 | 통합 대상 또는 비고 |
|---|---|---|---|
| P01 | S | 상세 작성 완료 | Thread 01 기초 벡터 연산·회귀 테스트 |
| P02 | S | 상세 작성 완료 | Thread 02 구·평면, Thread 05 선형 최근접 탐색, Thread 08 tie 계약 |
| P03 | S | 상세 작성 완료 | Thread 03 원기둥 입력 검증, Thread 07 bounds와 연결 |
| P04 | A | 상세 작성 완료 | 프레임 재사용 최적화와 동치 테스트 포함 |
| P05 | S | 상세 작성 완료 | 개별 directive 파싱, near-zero fix, loader 검증 포함 |
| P06 | S | 상세 작성 완료 | Thread 03·08 lifecycle, 08-03 private mutation boundary 포함 |
| P07 | A | 상세 작성 완료 | 직접광·shadow·bias를 하나로 통합 |
| P08 | A | 상세 작성 완료 | optional material grammar, depth CLI, diffuse regression 포함 |
| P09 | S | 상세 작성 완료 | AABB 값·결합과 경계 테스트 포함 |
| P10 | A | 상세 작성 완료 | 구·평면·원기둥 bounds와 unbounded 분리 포함 |
| P11 | S | 상세 작성 완료 | BVH 저장소·build·traversal·linear differential oracle 포함 |
| P12 | A | 상세 작성 완료 | Thread 09 계측, Thread 10 benchmark, Thread 13 gate 포함 |
| P13 | S | 상세 작성 완료 | thread count, pixel/checksum/work/PPM byte 결정성 포함 |
| P14 | S | 상세 작성 완료 | exception injection과 resource 회수 포함 |
| P15 | S | 상세 작성 완료 | 차원·곱셈 overflow·short/excess pixel storage 포함 |
| P16 | S | 상세 작성 완료 | stream failure·replacement failure·temporary cleanup 포함 |
| P17 | A | 상세 작성 완료 | accel·threads·max-depth·checksum·exit code 포함 |

누락된 S/A 항목은 없다. B/C 항목은 이 인덱스의 설명 연습 목록과 각 대표 문제의 원본 확인 위치에서만 다룬다.

## 백지 구현 우선순위

1. **P09 AABB slab 교차** — 짧지만 경계 조건이 많아 기본기 확인 효율이 가장 높다.
2. **P02 HitRecord·구 교차 계약** — 이후 모든 도형과 BVH의 correctness 기준이다.
3. **P15 이미지 저장소 크기·invariant** — integer overflow와 메모리 안전을 함께 본다.
4. **P14 worker 실패 전파** — exception, stop, join, RAII를 한 번에 검증한다.
5. **P05 strict parser** — 외부 입력 검증, 오류 위치, full consumption을 확인한다.
6. **P13 tile scheduler와 통계 reduction** — data race 없이 결정적 결과를 만드는 능력을 본다.
7. **P16 failure-safe 파일 교체** — 트랜잭션적 I/O와 cleanup을 검증한다.
8. **P11 결정적 BVH build** — 자료구조·정렬·tie policy를 확인한다.
9. **P03 유한 원기둥 교차** — 분석적 기하와 후보 통합 능력을 본다.
10. **P06 Scene lifecycle** — 원본 상태와 파생 인덱스의 정합성을 구현한다.
11. **P17 CLI parser** — 작은 상태 기계와 범위·중복 검증을 연습한다.
12. **P04 카메라 프레임** — 벡터 기초와 좌표계 convention을 확인한다.
13. **P07 shadow segment** — epsilon과 유한 구간을 구현한다.
14. **P08 깊이 제한 반사** — 재귀 종료와 dispatch를 구현한다.
15. **P10 보수적 bounds** — false negative 없는 경계 계산을 연습한다.
16. **P01 안정적 정규화** — 짧은 구현이지만 설명과 반례가 중요하다.
17. **P12 benchmark harness** — 구현보다 실험 설계가 중심이므로 마지막에 연습한다.

## 설명 연습 우선순위

1. **P06** — 왜 BVH를 자동으로 항상 최신화하지 않고 invalidation·fallback·명시적 rebuild를 택했는가.
2. **P11** — 최적화 구조가 선형 기준과 동일한 hit·tie 결과를 보장하는 방법.
3. **P13** — 스케줄이 달라도 픽셀과 통계가 결정적인 이유, `memory_order_relaxed`가 충분한 범위.
4. **P16** — 파일 쓰기를 commit/rollback 문제로 보는 이유와 원자적 교체의 한계.
5. **P12** — wall-clock, deterministic workload, checksum을 함께 보고해야 하는 이유.
6. **P05** — parser가 내부 객체 생성 전에 어떤 불변식을 확립해야 하는가.
7. **P02** — `frontFace`와 항상 입사 반대 방향인 normal이 셰이딩 코드를 단순하게 만드는 이유.
8. **P10** — tight bounds보다 conservative bounds가 우선인 이유.
9. **P14** — worker 예외를 즉시 던질 수 없고 join 이후 rethrow해야 하는 이유.
10. **P17** — usage error와 runtime error를 다른 exit code로 분리하는 API 의미.
11. **Thread 13 B 항목** — cross-platform compiler matrix, sanitizer job, timeout, 실패 로그 artifact, 안전한 clean 경계.

## 한 문제로 통합한 Thread 묶음

| 대표 문제 | 통합한 Thread·기록 | 통합 이유 |
|---|---|---|
| P01 안정적 정규화 | 01 수학 구현·회귀 + 03 near-zero parser + 04 camera frame | 같은 epsilon·norm 계약이 입력 검증과 좌표계 구성까지 전파된다. |
| P02 hit 계약 | 02 HitRecord·구·평면 + 05 선형 최근접 + 08 equal-distance 정책 | 교차 함수, aggregate 탐색, 가속 탐색이 같은 최근접·법선 계약을 공유한다. |
| P03 원기둥 교차 | 02 side·cap + 03 cylinder directive + 07 cylinder bounds | 기하 유효성, 교차, 가속 경계가 같은 원기둥 표현을 사용한다. |
| P05 strict parser | 03 numeric/vector/directive/loader + 01 수치 상수 + 12 CLI numeric parsing | 외부 문자열을 검증된 내부 값으로 바꾸는 공통 역량이다. |
| P06 Scene lifecycle | 03 acceleration build boundary + 08 unique ownership·mutation boundary·fallback | 원본 도형과 파생 BVH의 정합성을 하나의 상태 기계로 설명해야 한다. |
| P08 material dispatch | 03 optional material grammar + 06 metal reflection + 12 max-depth option | 입력 문법, 런타임 dispatch, 종료 정책이 한 기능의 end-to-end 계약이다. |
| P11 BVH correctness | 07 AABB·bounds + 08 build·traversal·tie + 10 workload evidence | pruning의 전제부터 결과 동치와 성능 증거까지 한 체인이다. |
| P12 performance evidence | 09 RenderStats + 10 benchmark + 13 verification gate | 계측 정의, 반복 측정, 자동 gate가 함께 있어야 수치가 설명 가능하다. |
| P13 parallel determinism | 09 tiled workers·thread count·동치 테스트 + 13 PPM byte gate | 구현의 race-free 성질과 프로세스 수준 결과 동치를 함께 검증한다. |
| P15 image invariant | 11 allocation/index overflow·validate + 09 renderer write pattern + 13 sanitizer gate | 크기 계산, 사용 지점, 동적 검사가 같은 메모리 안전 경계를 이룬다. |
| P16 failure-safe output | 11 stream/path writer·temporary replacement + 13 failure regression | 정상 쓰기보다 실패 시 기존 대상과 임시 자원을 어떻게 보존하는지가 핵심이다. |
| P17 CLI contract | 06 max depth + 09 thread count + 12 option parser·exit code + 13 CLI gate | 내부 설정과 외부 프로세스 API를 한 계약으로 연습한다. |
