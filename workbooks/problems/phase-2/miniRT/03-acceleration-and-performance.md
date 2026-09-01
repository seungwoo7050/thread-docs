# 가속 구조·성능 증거 워크북

이 문서는 P09~P12를 다룬다. AABB 교차와 도형 bounds에서 시작해 결정적 BVH 구축·순회, 마지막으로 성능 주장을 재현 가능한 증거로 만드는 방법까지 연결한다. 백지 구현은 각 알고리즘의 핵심 invariant만 남기고 원본 코드나 정답에 가까운 의사코드는 제공하지 않는다.

---

<a id="p09"></a>
## [Thread 07 / `feat(accel): ray-box slab 교차 구현`] AABB slab 교차와 닫힌 구간 계약

### 면접 질문

축 정렬 경계 상자와 광선의 교차를 slab 방식으로 계산하는 원리를 설명해 보세요. 방향 성분이 0인 축, 음수인 축, 상자 경계에 정확히 닿는 광선, 이미 제한된 `[tMin, tMax]` 구간을 각각 어떻게 처리해야 합니까?

꼬리 질문:

- 방향이 음수일 때 두 교차 파라미터의 순서를 바꿔야 하는 이유는 무엇입니까?
- `far < near`와 `far <= near` 중 어느 조건을 miss로 쓰느냐가 경계 접촉에 어떤 영향을 줍니까?
- 방향이 정확히 0일 때 나눗셈 대신 origin이 slab 안인지 확인해야 하는 이유는 무엇입니까?
- BVH 순회를 위해 상자 진입 `t`를 선택적으로 반환하면 무엇에 쓸 수 있습니까?
- `1 / direction`을 미리 계산하는 최적화에는 어떤 특별값 문제가 있습니까?
- 유효하지 않은 AABB는 어디에서 거부해야 합니까?

### 30초 모범 답변

각 축의 두 평면 사이에 광선이 머무는 `t` 구간을 구하고, 세 축 구간과 호출자가 준 `[tMin, tMax]`의 교집합이 비어 있는지 확인합니다. 방향이 음수면 두 평면의 `t` 순서가 뒤집히므로 정렬하고, 방향이 0이면 origin이 그 축의 slab 밖일 때만 즉시 miss로 처리합니다. 경계 접촉을 hit로 인정하려면 교집합이 한 점인 경우를 남겨 두어 `far < near`일 때만 miss로 봅니다. 최종 near 값은 BVH에서 가까운 child를 먼저 순회하고 현재 최단 hit보다 먼 node를 prune하는 데 사용할 수 있습니다.

### 답변 핵심 키워드

`slab interval`, `구간 교집합`, `negative direction swap`, `parallel axis`, `boundary hit`, `far < near`, `entry t`, `BVH near-first`, `invalid box`

### 백지 구현

**구현 목표**

유효한 AABB와 광선의 교차 여부를 주어진 `t` 구간 안에서 계산하고, 요청된 경우 최초 진입 거리를 반환한다.

**인터페이스**

```cpp
struct Aabb {
    Vec3 minimum;
    Vec3 maximum;

    bool isValid() const;

    bool intersect(const Ray& ray,
                   double tMin,
                   double tMax,
                   double* entry = nullptr) const;
};
```

**입력과 출력**

- 입력: AABB, 광선, 유효 `t` 구간, 선택적 entry output.
- 출력: 구간 내 상자 교차 여부. 성공하고 `entry`가 있으면 교집합의 시작 `t`.

**반드시 만족해야 할 조건**

- 유효하지 않은 상자는 miss로 처리한다.
- 기존 `[tMin, tMax]`를 각 축의 구간과 계속 교차한다.
- 방향 부호와 관계없이 같은 기하 결과를 낸다.
- 방향 성분이 0인 축에서 0으로 나누지 않는다.
- 상자 경계에 접하는 한 점 교차의 정책이 명시적이어야 한다.
- 실패 시 `entry`를 성공 값처럼 사용하게 만들지 않는다.

**경계 조건**

- 상자 정면·후면에서 들어오는 광선.
- 한 개 이상의 음수 방향 성분.
- 축과 평행하면서 slab 내부에 있는 광선.
- 축과 평행하면서 slab 외부에 있는 광선.
- 상자 면·모서리·꼭짓점에 닿는 광선.
- 광선 원점이 상자 내부에 있음.
- 호출자의 `tMin`이 실제 진입점보다 큼.
- 퇴화하거나 역전된 minimum/maximum.

**실패 조건**

- 음수 방향에서 near/far가 뒤집힌 채 사용된다.
- 평행 축에서 무한대·NaN 비교에 결과를 맡긴다.
- 한 축의 miss를 다른 축 계산이 덮어쓴다.
- 호출자가 준 `t` 구간을 무시한다.
- 접촉 정책이 테스트마다 달라진다.

**제약**

- 회전 상자는 다루지 않는다.
- SIMD나 역방향 캐시는 구현하지 않는다.
- 시간 복잡도 O(1), 추가 공간 O(1)을 유지한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 양의 z 방향 정면 hit의 entry가 기대한 값인가.
- [ ] 같은 광선을 반대 방향에서 쏘아도 올바르게 hit하는가.
- [ ] 평행하면서 상자 내부 slab에 있는 축을 통과시키는가.
- [ ] 평행하면서 slab 밖인 광선을 즉시 거부하는가.
- [ ] 면 경계 위 origin과 경계 접촉을 정한 정책대로 처리하는가.
- [ ] 상자 내부에서 시작하면 entry가 호출 구간 시작과 일관되는가.
- [ ] 좁은 `[tMin, tMax]`가 실제 교차를 제외할 수 있는가.
- [ ] invalid AABB가 안전하게 거부되는가.
- [ ] 실패 시 entry에 의존하지 않는가.

### 구현 후 설명할 것

1. 세 축 interval의 교집합으로 보는 관점.
2. 방향 0을 별도 분기한 이유와 비교 정책.
3. 경계 한 점 접촉을 hit로 볼지 정한 기준.
4. entry 값을 BVH 순서와 pruning에 활용하는 방법.
5. 미리 역방향을 계산하는 최적화의 장단점.

### 원본 확인 위치

- Thread 07
- `feat(accel): AABB 값과 결합 연산 구현`
- `feat(accel): ray-box slab 교차 구현`
- `test(accel): AABB와 도형 경계 계산 검증`
- `include/ray/accel.hpp`
- `src/accel.cpp`
- `tests/core_tests.cpp`
- `Aabb`, `Aabb::isValid`, `Aabb::centroid`, `Aabb::intersect`, `surroundingBox`
- 관련 Thread: 08의 BVH build/traversal, 10의 `aabbTests` 계측

---

<a id="p10"></a>
## [Thread 07 / `feat(accel): 원기둥의 보수적 bounds 계산 추가`] 도형 bounds의 보수성 invariant

### 면접 질문

BVH용 도형 bounds에서 "정확히 타이트함"보다 "실제 도형을 절대 누락하지 않음"이 우선인 이유를 설명해 보세요. 구, 무한 평면, 임의 축 유한 원기둥은 각각 어떤 bounds 계약을 가져야 합니까?

꼬리 질문:

- bounds가 실제 도형보다 조금 크면 어떤 비용이 생기고, 조금 작으면 어떤 correctness 문제가 생깁니까?
- 무한 평면에 임의의 큰 AABB를 주는 방식이 위험한 이유는 무엇입니까?
- 임의 축 원기둥의 한 좌표축 방향 extent를 생각할 때 축 방향 길이와 반지름 기여를 어떻게 분해할 수 있습니까?
- cap과 옆면을 모두 포함해야 하는 이유는 무엇입니까?
- 계산된 최소·최대 값을 바깥쪽으로 `nextafter` 하는 목적은 무엇입니까?
- epsilon을 크게 늘리는 것과 ULP 단위로 확장하는 것의 trade-off는 무엇입니까?

### 30초 모범 답변

BVH bounds는 broad phase이므로 false positive는 추가 교차 검사만 만들지만 false negative는 실제 hit를 영구히 놓칩니다. 구는 중심에서 반지름만큼 각 축으로 확장하면 되고, 무한 평면은 유한 bounds가 없다고 표시해 BVH 밖에서 별도로 검사해야 합니다. 임의 축 원기둥은 각 세계 축에 대해 중심축 반길이의 투영과 축에 수직인 원 반지름의 투영을 합쳐 전체 extent를 구하고 cap까지 포함해야 합니다. 계산 오차로 실제 표면이 경계 밖에 나가지 않도록 최소·최대를 바깥 방향의 인접 표현값으로 확장하면 타이트함을 거의 잃지 않으면서 보수성을 강화할 수 있습니다.

### 답변 핵심 키워드

`broad phase`, `false positive 허용`, `false negative 금지`, `optional bounds`, `unbounded plane`, `axis projection`, `radial projection`, `cap 포함`, `outward nextafter`, `tightness vs robustness`

### 백지 구현

**구현 목표**

구·평면·임의 축 유한 원기둥의 `bounds()`를 구현한다. 원기둥 축은 생성 시 정규화되어 있다고 가정한다.

**인터페이스**

```cpp
class Shape {
public:
    virtual std::optional<Aabb> bounds() const = 0;
};

class Sphere : public Shape {
public:
    std::optional<Aabb> bounds() const override;
};

class Plane : public Shape {
public:
    std::optional<Aabb> bounds() const override;
};

class Cylinder : public Shape {
public:
    std::optional<Aabb> bounds() const override;
};
```

**입력과 출력**

- 입력: 각 도형의 중심·축·반지름·높이 등 이미 검증된 기하 값.
- 출력: 실제 도형 전체를 포함하는 유효 AABB 또는 유한 bounds가 없음을 뜻하는 `std::nullopt`.

**반드시 만족해야 할 조건**

- 구의 bounds는 모든 축에서 표면을 포함해야 한다.
- 무한 평면은 유한 상자를 가장하지 않는다.
- 원기둥 bounds는 옆면과 두 cap을 모두 포함한다.
- 임의 방향의 단위 축을 처리해야 한다.
- 부동소수점 반올림 때문에 실제 표면이 상자 밖으로 나갈 가능성을 줄여야 한다.
- 반환한 AABB는 `isValid()`를 만족해야 한다.

**경계 조건**

- 세계 x·y·z 축과 나란한 원기둥.
- 모든 축 성분이 섞인 원기둥.
- 매우 작지만 유효한 반지름과 높이.
- 음수 축 성분을 가진 원기둥.
- 구와 원기둥의 표면점이 bounds 경계에 정확히 놓임.
- 무한 도형과 유한 도형이 같은 장면에 있음.

**실패 조건**

- cap을 빠뜨려 원기둥 끝부분을 포함하지 못한다.
- 축이 기울어진 경우 단순 `center ± (radius, height/2, radius)`를 사용한다.
- 평면에 임의 크기의 유한 상자를 부여한다.
- 계산 오차로 실제 표면보다 작은 상자를 반환한다.
- 너무 큰 임의 epsilon으로 bounds가 과도하게 부풀어 BVH 효과를 잃는다.

**제약**

- 도형 교차 함수는 구현하지 않는다.
- 원기둥 축 정규화 검증은 생성자나 parser가 맡는다고 가정한다.
- 타이트한 최소 AABB가 이상적이지만, correctness를 해치지 않는 작은 보수 확장을 허용한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 구의 여섯 극점이 상자 내부 또는 경계에 있는가.
- [ ] 평면이 `nullopt`를 반환하는가.
- [ ] 축 정렬 원기둥의 예상 bounds가 나오는가.
- [ ] 임의 축 원기둥의 cap 중심과 원 둘레 대표점이 모두 포함되는가.
- [ ] 축 부호를 뒤집어도 같은 기하 bounds가 나오는가.
- [ ] 무작위 표면 샘플을 생성했을 때 하나도 상자 밖으로 나가지 않는가.
- [ ] 최소·최대가 유한하고 순서가 올바른가.
- [ ] 보수 확장이 불필요하게 큰 상자를 만들지 않는가.
- [ ] 계산은 O(1) 시간과 O(1) 공간인가.

### 구현 후 설명할 것

1. broad-phase bounds에서 false positive와 false negative의 비대칭성.
2. 무한 평면을 별도 목록으로 보내는 설계.
3. 원기둥 extent를 세계 축별 투영으로 계산한 논리.
4. cap과 옆면 중 최대 extent를 취하는 이유.
5. 고정 epsilon과 `nextafter` 기반 바깥 확장의 차이.

### 원본 확인 위치

- Thread 07
- `feat(accel): 도형 경계 계약과 구·평면 bounds 추가`
- `feat(accel): 원기둥의 보수적 bounds 계산 추가`
- `test(accel): AABB와 도형 경계 계산 검증`
- `include/ray/geometry.hpp`
- `src/geometry.cpp`
- `tests/core_tests.cpp`
- `Shape::bounds`, `Sphere::bounds`, `Plane::bounds`, `Cylinder::bounds`
- 관련 Thread: 02의 도형 기하, 08의 bounded/unbounded 분리와 BVH 구축

---

<a id="p11"></a>
## [Thread 08 / `feat(accel): 결정적 중앙 분할 BVH 구축 구현`] 결정적 BVH 구축·최근접 순회·tie 동치

### 면접 질문

BVH를 중앙 분할로 구축할 때 결과가 실행마다 달라지지 않게 하려면 정렬 축, 동일 centroid 처리, leaf 조건을 어떻게 정의해야 합니까? 또 BVH 최근접 순회가 선형 탐색과 픽셀 단위로 동일한 결과를 내려면 어떤 hit tie 정책과 순회 invariant가 필요합니까?

꼬리 질문:

- 가장 긴 centroid extent 축을 고르는 단순 heuristic의 장단점은 무엇입니까?
- centroid가 모두 같은 경우에도 recursion이 종료되게 하려면 무엇이 필요합니까?
- 연속 `nodes` 저장소와 별도 primitive-index 배열을 쓰는 이유는 무엇입니까?
- 순회에서 가까운 child를 먼저 방문하면 correctness가 아니라 성능에 어떤 이점이 있습니까?
- 현재 node entry가 이미 찾은 `closest`보다 크면 왜 prune할 수 있습니까?
- 동일 `t`의 두 도형에서 선형 순회와 BVH 순회 순서가 다르더라도 같은 도형을 선택하려면 어떻게 해야 합니까?
- unbounded 도형은 BVH 결과와 언제 합쳐야 합니까?

### 30초 모범 답변

구축은 leaf 최대 개수를 정하고, centroid bounds의 가장 긴 축을 선택한 뒤 centroid와 원래 shape index를 포함한 안정적 총순서로 정렬해 중앙에서 나눕니다. shape index까지 tie-breaker로 쓰면 동일 centroid에서도 결과가 결정적이고, 중앙 분할은 매 단계 범위를 줄여 종료됩니다. 순회는 명시적 stack과 AABB entry를 사용해 가까운 child를 먼저 검사하고, entry가 현재 `closest`보다 먼 node는 버립니다. 실제 도형 hit를 갱신할 때는 거리 우선, 동일 거리에서는 명시한 shape-index 정책을 적용하므로 순회 순서가 달라도 선형 모드와 같은 결과가 납니다. bounds가 없는 도형은 별도로 검사해 최종 closest 경쟁에 참여시킵니다.

### 답변 핵심 키워드

`centroid bounds`, `largest extent axis`, `stable total order`, `shape-index tie`, `median split`, `bounded leaf size`, `contiguous nodes`, `explicit stack`, `near-first`, `closest pruning`, `mode equivalence`, `unbounded merge`

### 백지 구현

**구현 목표**

주어진 `BvhPrimitive` 배열을 최대 leaf 크기 이하의 결정적 BVH로 만드는 `buildNode`를 구현한다. 순회 코드는 구현하지 않지만, 순회가 사용할 수 있는 node·index invariant를 만족해야 한다.

**인터페이스**

```cpp
struct BvhPrimitive {
    std::uint32_t shapeIndex;
    Aabb bounds;
};

struct BvhNode {
    Aabb bounds;
    std::uint32_t left;
    std::uint32_t right;
    std::uint32_t first;
    std::uint32_t count;

    bool isLeaf() const;
};

class Bvh {
public:
    void build(std::vector<BvhPrimitive> primitives);

private:
    std::uint32_t buildNode(std::vector<BvhPrimitive>& primitives,
                            std::uint32_t first,
                            std::uint32_t last);

    std::vector<BvhNode> nodes_;
    std::vector<std::uint32_t> primitiveIndices_;
};
```

**입력과 출력**

- 입력: 유효 bounds와 원래 장면 index를 가진 bounded primitive 배열.
- 출력: root가 index 0에서 시작하는 연속 node 저장소와 모든 primitive를 정확히 한 번 담는 index 배열.

**반드시 만족해야 할 조건**

- 빈 입력은 빈 BVH를 만든다.
- 모든 primitive가 정확히 한 leaf에 들어간다.
- leaf의 primitive 수는 정한 최대값을 넘지 않는다.
- 내부 node는 비어 있지 않은 두 child를 가진다.
- 부모 bounds는 두 child의 bounds를 모두 포함한다.
- 동일 입력은 항상 같은 node·primitive 순서를 만든다.
- centroid가 같은 primitive도 원래 `shapeIndex`로 총순서를 정할 수 있어야 한다.
- recursion마다 범위가 엄격히 줄어 종료해야 한다.

**경계 조건**

- primitive 0개, 1개, leaf 최대값과 같은 개수.
- leaf 최대값보다 하나 많은 개수.
- 홀수 개수.
- centroid가 모두 동일함.
- 한 축 extent만 큼.
- 모든 축 extent가 0이거나 동일함.
- shape index 입력 순서가 뒤섞여 있음.

**실패 조건**

- 동일 centroid 정렬 결과가 라이브러리·실행마다 달라진다.
- split 위치가 한쪽 끝이라 빈 child나 무한 recursion이 생긴다.
- primitive가 중복되거나 누락된다.
- node vector 재할당 뒤 저장한 reference를 계속 사용한다.
- leaf의 `first/count`가 최종 primitive-index 배열과 맞지 않는다.
- parent bounds가 child를 완전히 포함하지 못한다.

**제약**

- SAH는 구현하지 않는다.
- 최대 leaf 크기는 작은 고정값으로 둔다.
- 재귀 또는 명시적 build stack 중 하나를 선택할 수 있다.
- 구축 시간은 일반적으로 O(n log n)을 목표로 한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 빈 입력에서 node와 index 저장소가 모두 비는가.
- [ ] 하나의 primitive가 하나의 leaf로 보존되는가.
- [ ] 최대 leaf 크기 경계에서 불필요한 분할이 없는가.
- [ ] 최대값을 넘으면 두 비어 있지 않은 하위 범위로 나뉘는가.
- [ ] 홀수 개수가 전부 한 번씩 배치되는가.
- [ ] 동일 centroid 입력을 여러 번 섞어 넣어도 결과가 결정적인가.
- [ ] 모든 leaf의 `first/count` 범위가 유효한가.
- [ ] 모든 primitive index의 다중집합이 입력과 같은가.
- [ ] 각 parent bounds가 재귀적으로 모든 descendant를 포함하는가.
- [ ] 최악의 퇴화 입력에서도 recursion이 끝나는가.
- [ ] 선형 탐색과 BVH 탐색의 hit, shape, 픽셀, checksum을 differential test할 계획이 있는가.

### 구현 후 설명할 것

1. 중앙 분할을 택한 구현 단순성과 SAH 대비 품질 trade-off.
2. centroid·shape index 총순서가 결정성에 미치는 영향.
3. 연속 node·primitive index 표현의 cache locality와 포인터 안정성.
4. 최근접 순회에서 near-first와 `closest` pruning이 작동하는 방식.
5. 순회 순서와 무관한 equal-distance tie 계약이 필요한 이유.

### 원본 확인 위치

- Thread 08
- `feat(accel): BVH node와 연속 저장소 구성`
- `feat(accel): 결정적 중앙 분할 BVH 구축 구현`
- `feat(accel): 선형·BVH 탐색 모드 계약 연결`
- `feat(accel): 결정적 BVH 최근접 순회 구현`
- `include/ray/accel.hpp`
- `src/accel.cpp`
- `include/ray/scene.hpp`
- `src/scene.cpp`
- `tests/accel_tests.cpp`
- `BvhPrimitive`, `BvhNode`, `BvhNode::isLeaf`, `Bvh`, `Bvh::build`, `Bvh::buildNode`, `Bvh::clear`, `AccelMode`, `Scene::intersect`
- 관련 Thread: 07의 AABB와 bounds, 08의 lifecycle, 09의 실행 모드 결정성, 10의 선형·BVH 성능 비교

---

<a id="p12"></a>
## [Thread 10 / `perf(benchmark): 반복 측정과 결정성 보고 구성`] 재현 가능한 성능 증거와 작업량 계측

### 면접 질문

BVH가 빨라졌다는 주장을 wall-clock 한 번의 측정만으로 입증하면 왜 부족합니까? 이 프로젝트에서 고정 workload, warm-up, 반복 측정의 median, checksum, 작업량 counter, 단일 thread, schema·환경 기록을 함께 둔 이유를 설명해 보세요.

꼬리 질문:

- 시간 측정과 `primitiveTests`·`aabbTests` 같은 결정적 작업량 지표는 각각 무엇을 설명합니까?
- 가장 빠른 값이나 평균 대신 median을 쓴 이유는 무엇입니까?
- 선형과 BVH의 checksum이 다르면 성능 수치를 왜 폐기해야 합니까?
- benchmark에 자동 thread 선택을 두면 비교 해석이 어려워지는 이유는 무엇입니까?
- warm-up 한 번으로 충분하다고 일반화할 수 있습니까?
- `primitiveTests` 감소율을 gate로 쓰고 속도 배율은 보고만 하는 설계의 장점은 무엇입니까?
- reference JSON에 compiler·architecture·logical threads를 남겨야 하는 이유는 무엇입니까?

### 30초 모범 답변

Wall-clock은 스케줄링, 주파수, 캐시와 백그라운드 작업에 흔들리므로 한 번의 시간만으로 알고리즘 개선을 주장하기 어렵습니다. 그래서 같은 조밀 장면과 동일 설정을 고정하고, 한 번 예열한 뒤 여러 번 측정해 median을 보고합니다. 각 반복의 checksum과 광선·primitive·AABB counter가 같아야 같은 일을 했다고 볼 수 있고, 선형과 BVH checksum도 같아야 correctness를 보존한 비교입니다. thread 수를 1로 고정해 가속 구조 효과와 병렬 효과를 분리하고, 환경과 schema를 함께 기록합니다. 시간은 관찰값이고 primitive-test 비율은 더 안정적인 구조적 근거라서, 후자를 성능 gate로 삼기 쉽습니다.

### 답변 핵심 키워드

`fixed workload`, `warm-up`, `repeated samples`, `median`, `wall-clock noise`, `checksum equivalence`, `deterministic counters`, `single thread`, `algorithm isolation`, `schema version`, `environment metadata`, `structural gate`

### 백지 구현

**구현 목표**

이미 제공된 `render(scene, mode)`를 반복 호출해 각 모드의 대표 sample을 고르고, 결과 결정성을 검증하는 축소 benchmark harness를 구현한다.

**인터페이스**

```cpp
struct Sample {
    double milliseconds;
    RenderStats stats;
    std::string checksum;
};

Sample render(const Scene& scene, AccelMode mode);

Sample measure(const Scene& scene,
               AccelMode mode,
               int warmupRuns,
               int measuredRuns);
```

**입력과 출력**

- 입력: 고정 장면, 가속 모드, warm-up 횟수, 측정 횟수.
- 출력: 시간 기준 median에 해당하는 `Sample`.
- 실패: 반복 결과의 checksum이나 결정적 작업량이 다르면 오류를 보고한다.

**반드시 만족해야 할 조건**

- warm-up 결과는 대표 sample 선택에 포함하지 않는다.
- 측정 횟수는 양수여야 한다.
- measured sample은 시간을 기준으로 정렬해 median을 선택한다.
- 모든 measured run의 checksum이 같아야 한다.
- primary, secondary, shadow, primitive, AABB 작업량이 반복마다 같아야 한다.
- 선형·BVH 비교는 동일한 장면과 설정, 특히 thread 수를 사용해야 한다.
- 성능 비교 전에 두 모드의 checksum 동치를 별도로 검증할 수 있어야 한다.

**경계 조건**

- measured run이 1개.
- 홀수·짝수 measured run. 짝수에서 어떤 median 정의를 쓸지 명시.
- 측정 시간은 다르지만 checksum과 작업량은 같음.
- checksum은 같지만 작업량이 달라짐.
- 한 run만 비정상적으로 느림.
- 매우 짧아 timer 분해능 영향을 받는 workload.

**실패 조건**

- 가장 빠른 sample만 선택해 편향을 만든다.
- correctness가 다른 두 모드를 시간만 비교한다.
- thread 수나 깊이 등 설정이 모드별로 다르다.
- 반복마다 작업량이 바뀌는데도 시간 median만 보고한다.
- 환경 정보 없는 단일 숫자를 보편적 기준처럼 기록한다.

**제약**

- JSON serializer를 구현할 필요는 없다.
- 통계 검정이나 confidence interval은 요구하지 않는다.
- benchmark 함수 자체가 렌더러 상태를 변경하지 않는다고 가정한다.
- O(k log k) 정렬 또는 동등한 median 선택을 사용할 수 있다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] warm-up 호출 수와 measured 호출 수가 구분되는가.
- [ ] measured run 1개에서 그 sample을 반환하는가.
- [ ] outlier 하나가 대표값을 과도하게 바꾸지 않는가.
- [ ] checksum 불일치 시 즉시 비교를 거부하는가.
- [ ] 각 결정적 counter의 불일치를 탐지하는가.
- [ ] median sample의 시간과 stats/checksum이 같은 한 sample에서 함께 오는가.
- [ ] 선형·BVH가 동일 configuration으로 실행되는가.
- [ ] 결과 보고에 scene, 크기, 도형 수, threads, depth, tile size, 반복 수를 남길 수 있는가.
- [ ] reference 값과 현재 값을 환경 차이 없이 직접 비교하지 않는가.
- [ ] 측정 harness의 추가 메모리가 O(k)이고 렌더링 결과 자체를 불필요하게 모두 보관하지 않는가.

### 구현 후 설명할 것

1. 시간과 작업량 counter를 함께 사용한 이유.
2. median과 warm-up이 줄이는 노이즈와 줄이지 못하는 노이즈.
3. correctness equivalence를 성능 비교보다 먼저 검사하는 원칙.
4. 단일 thread로 알고리즘 효과를 분리한 판단.
5. 환경·schema 기록과 자동 gate를 어디까지 신뢰할 수 있는지.

### 원본 확인 위치

- Thread 09의 `perf(render): 광선과 교차 작업량 계측 추가`
- Thread 10
- `perf(benchmark): 조밀 장면 기준 workload 추가`
- `perf(benchmark): 반복 측정과 결정성 보고 구성`
- `perf(benchmark): 선형 탐색과 BVH 작업량 비교`
- `perf(benchmark): 측정 schema와 가속 기준 검증 고정`
- `perf(benchmark): 참조 측정값 기록`
- `include/ray/renderer.hpp`
- `src/renderer.cpp`
- `src/scene.cpp`
- `src/shading.cpp`
- `benchmarks/render_benchmark.cpp`
- `benchmarks/reference.json`
- `RenderStats`, `Sample`, `render`, `measure`, `printResult`
- 관련 Thread: 08의 BVH 동치·primitive 감소, 09의 worker별 통계 합산과 실행 모드 결정성, 13의 재현 가능한 검증 gate
