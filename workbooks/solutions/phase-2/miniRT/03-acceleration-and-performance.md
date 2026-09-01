# 가속 구조·성능 증거 워크북

이 문서는 P09~P12를 다룬다. AABB 교차와 도형 bounds에서 시작해 결정적 BVH 구축·순회, 마지막으로 성능 주장을 재현 가능한 증거로 만드는 방법까지 연결한다. 백지 구현은 각 알고리즘의 핵심 invariant만 남기고 원본 코드나 정답에 가까운 의사코드는 제공하지 않는다.

---

<a id="p09"></a>
## [Thread 07 / `feat(accel): ray-box slab 교차 구현`] AABB slab 교차와 닫힌 구간 계약

### 면접 질문

축 정렬 경계 상자와 광선의 교차를 slab 방식으로 계산하는 원리를 설명해 보세요. 방향 성분이 0인 축, 음수인 축, 상자 경계에 정확히 닿는 광선, 이미 제한된 `[tMin, tMax]` 구간을 각각 어떻게 처리해야 합니까?

꼬리 질문:

- 방향이 음수일 때 두 교차 파라미터의 순서를 바꿔야 하는 이유는 무엇입니까?
  - **모범답변:** `slabMin`에서 계산한 `t`가 항상 진입점이라는 보장은 방향이 양수일 때만 성립합니다. 방향이 음수면 두 값의 순서가 뒤집히므로 작은 값을 near, 큰 값을 far로 재정렬해야 합니다.
- `far < near`와 `far <= near` 중 어느 조건을 miss로 쓰느냐가 경계 접촉에 어떤 영향을 줍니까?
  - **모범답변:** 원본은 `far < near`만 miss로 처리해 두 값이 같은 한 점 교차를 hit로 인정합니다. `<=`를 쓰면 상자의 면·모서리·꼭짓점에 정확히 접하는 경우를 제외합니다.
- 방향이 정확히 0일 때 나눗셈 대신 origin이 slab 안인지 확인해야 하는 이유는 무엇입니까?
  - **모범답변:** 그 축 좌표는 광선 전체에서 변하지 않으므로 origin이 slab 밖이면 영원히 들어올 수 없고, 안이면 그 축은 `t` 구간을 제한하지 않습니다. 0으로 나눠 생기는 무한대·NaN 규칙에 의존할 필요가 없습니다.
- BVH 순회를 위해 상자 진입 `t`를 선택적으로 반환하면 무엇에 쓸 수 있습니까?
  - **모범답변:** 두 child 중 entry가 작은 쪽을 먼저 stack에서 처리하고, 나중에 그 entry가 이미 찾은 `closest`보다 크면 node 전체를 prune할 수 있습니다.
- `1 / direction`을 미리 계산하는 최적화에는 어떤 특별값 문제가 있습니까?
  - **모범답변:** 0 방향은 부호 있는 무한대가 되고, `0 * infinity` 같은 계산은 NaN이 될 수 있습니다. 원점이 경계에 있는 경우 비교도 까다로워지므로 0 축의 명시적 slab 분기와 특별값 테스트가 필요합니다.
- 유효하지 않은 AABB는 어디에서 거부해야 합니까?
  - **모범답변:** 원본 `Aabb::intersect`가 먼저 `isValid()`를 검사해 자체적으로 miss를 반환하고, `Scene::buildAcceleration`도 invalid bounds를 unbounded 경로로 분류합니다. 일반적으로 생성 경계에서 막되 소비 함수도 안전한 계약을 갖는 것이 좋습니다.

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
bool Aabb::isValid() const {
    return minimum.x <= maximum.x &&
           minimum.y <= maximum.y &&
           minimum.z <= maximum.z;
}

bool Aabb::intersect(const Ray& ray, double tMin, double tMax,
                     double* entry) const {
    if (!isValid()) return false;

    double nearValue = tMin;
    double farValue = tMax;
    const double origins[] = {ray.origin.x, ray.origin.y, ray.origin.z};
    const double directions[] = {
        ray.direction.x, ray.direction.y, ray.direction.z
    };
    const double mins[] = {minimum.x, minimum.y, minimum.z};
    const double maxs[] = {maximum.x, maximum.y, maximum.z};

    for (int axis = 0; axis < 3; ++axis) {
        if (directions[axis] == 0.0) {
            // 평행 축은 origin이 slab 안이면 구간을 제한하지 않는다.
            if (origins[axis] < mins[axis] || origins[axis] > maxs[axis])
                return false;
            continue;
        }
        double first = (mins[axis] - origins[axis]) / directions[axis];
        double second = (maxs[axis] - origins[axis]) / directions[axis];
        if (first > second) std::swap(first, second);
        nearValue = std::max(nearValue, first);
        farValue = std::min(farValue, second);
        if (farValue < nearValue) return false; // 한 점 접촉은 hit다.
    }

    if (entry) *entry = nearValue; // 성공한 경우에만 output을 기록한다.
    return true;
}
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
   - **모범답변:** 각 축 slab 안에 광선이 있는 `t` 구간을 구하고 호출자의 초기 `[tMin,tMax]`에 차례로 교차합니다. 세 축을 처리한 뒤 구간이 남아 있으면 그 `t`에서 모든 축 조건을 동시에 만족합니다.
2. 방향 0을 별도 분기한 이유와 비교 정책.
   - **모범답변:** 방향 0에서는 좌표가 변하지 않으므로 origin이 닫힌 slab `[min,max]` 밖일 때만 miss입니다. 이는 0 나눗셈과 특별값 전파 없이 기하 의미를 직접 표현합니다.
3. 경계 한 점 접촉을 hit로 볼지 정한 기준.
   - **모범답변:** 프로젝트는 닫힌 구간 계약을 사용해 `far == near`를 hit로 둡니다. 그래서 miss 조건은 `far < near`이고 도형 경계의 접촉을 broad phase에서 누락하지 않습니다.
4. entry 값을 BVH 순서와 pruning에 활용하는 방법.
   - **모범답변:** entry가 작은 child를 먼저 방문해 가까운 실제 hit를 일찍 찾습니다. 그 뒤 stack의 node entry가 `closest`보다 크면 그 node 안의 모든 hit도 더 멀므로 버릴 수 있습니다.
5. 미리 역방향을 계산하는 최적화의 장단점.
   - **모범답변:** 여러 상자를 검사할 때 나눗셈을 줄일 수 있지만 0, 부호 있는 0, 무한대와 NaN 처리가 복잡해집니다. 현재 원본은 축마다 직접 나누어 단순하고 명시적인 계약을 택했습니다.

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
  - **모범답변:** 큰 bounds는 false positive로 node와 primitive 검사를 조금 늘릴 뿐입니다. 작은 bounds는 broad phase에서 실제 교차를 prune해 hit가 영구히 누락되므로 correctness를 깨뜨립니다.
- 무한 평면에 임의의 큰 AABB를 주는 방식이 위험한 이유는 무엇입니까?
  - **모범답변:** 어떤 유한 크기도 평면 전체를 포함하지 못해 그 밖의 hit를 누락합니다. 지나치게 크게 잡아도 대부분의 ray가 상자를 맞아 BVH 분할 효과가 사라지므로 원본은 `nullopt`와 별도 unbounded 목록을 사용합니다.
- 임의 축 원기둥의 한 좌표축 방향 extent를 생각할 때 축 방향 길이와 반지름 기여를 어떻게 분해할 수 있습니까?
  - **모범답변:** 단위 축 성분을 `a`라 하면 중심축 반높이의 투영은 `|a|*halfHeight`, 축에 수직인 원의 해당 세계축 투영은 `radius*sqrt(max(0,1-a*a))`입니다. 두 기여를 합쳐 좌표별 extent를 구합니다.
- cap과 옆면을 모두 포함해야 하는 이유는 무엇입니까?
  - **모범답변:** 닫힌 원기둥의 표면은 옆면뿐 아니라 두 원판도 포함합니다. 반올림 보수 확장 방식이 서로 달라질 수 있으므로 원본은 side와 cap extent를 각각 계산하고 큰 값을 선택합니다.
- 계산된 최소·최대 값을 바깥쪽으로 `nextafter` 하는 목적은 무엇입니까?
  - **모범답변:** 계산된 경계가 반올림으로 실제 표면보다 안쪽이 되는 것을 막기 위해 최소는 `-infinity`, 최대는 `+infinity` 방향의 바로 다음 표현값으로 옮깁니다. ULP 한 칸만 늘려 타이트함 손실을 최소화합니다.
- epsilon을 크게 늘리는 것과 ULP 단위로 확장하는 것의 trade-off는 무엇입니까?
  - **모범답변:** 고정 epsilon은 이해하기 쉽지만 좌표 scale에 따라 과하거나 부족합니다. `nextafter`는 표현값 크기에 맞춘 최소 확장이지만 앞선 기하 계산의 누적 오차까지 모두 흡수하지는 못해 원본은 계산 내부 epsilon과 최종 ULP 확장을 함께 사용합니다.

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
std::optional<Aabb> Sphere::bounds() const {
    const Vec3 extent{radius_, radius_, radius_};
    return Aabb{center_ - extent, center_ + extent};
}

std::optional<Aabb> Plane::bounds() const {
    return std::nullopt; // 무한 도형은 BVH 밖의 별도 경로에서 검사한다.
}

std::optional<Aabb> Cylinder::bounds() const {
    const double halfHeight = height_ * 0.5;
    const auto extentFor = [&](double axisComponent) {
        const double axial = std::fabs(axisComponent);
        const double radial = std::sqrt(
            std::max(0.0, 1.0 - axisComponent * axisComponent));
        const double side = axial * (halfHeight + kEpsilon) +
                            radius_ * radial;
        const double cap = axial * halfHeight +
            std::sqrt(radius_ * radius_ + kEpsilon) * radial;
        return std::max(side, cap);
    };

    const Vec3 extent{
        extentFor(axis_.x), extentFor(axis_.y), extentFor(axis_.z)
    };
    Vec3 min = center_ - extent;
    Vec3 max = center_ + extent;
    // 각 경계를 바깥쪽 한 ULP로 확장해 false negative를 피한다.
    min.x = std::nextafter(min.x, -std::numeric_limits<double>::infinity());
    min.y = std::nextafter(min.y, -std::numeric_limits<double>::infinity());
    min.z = std::nextafter(min.z, -std::numeric_limits<double>::infinity());
    max.x = std::nextafter(max.x, std::numeric_limits<double>::infinity());
    max.y = std::nextafter(max.y, std::numeric_limits<double>::infinity());
    max.z = std::nextafter(max.z, std::numeric_limits<double>::infinity());
    return Aabb{min, max};
}
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
   - **모범답변:** false positive는 좁은 단계의 실제 도형 교차를 추가로 실행할 뿐 결과는 같습니다. false negative는 해당 subtree를 건너뛰어 실제 hit를 복구할 기회가 없으므로 bounds는 보수적이어야 합니다.
2. 무한 평면을 별도 목록으로 보내는 설계.
   - **모범답변:** 평면은 유한 AABB 계약을 만족할 수 없으므로 `nullopt`를 반환합니다. Scene은 그 index를 별도로 저장해 BVH 순회 뒤에도 같은 최근접 비교에 참여시킵니다.
3. 원기둥 extent를 세계 축별 투영으로 계산한 논리.
   - **모범답변:** 각 세계축에 대해 중심축 선분 투영 `|a|*halfHeight`와 수직 원의 최대 투영 `radius*sqrt(1-a²)`를 합칩니다. 단위 축의 각 성분에 같은 공식을 적용해 임의 방향을 처리합니다.
4. cap과 옆면 중 최대 extent를 취하는 이유.
   - **모범답변:** 닫힌 표면의 두 부분을 모두 포함해야 하고 원본은 side 높이와 cap 반지름에 서로 다른 보수 epsilon을 적용합니다. 좌표별 최댓값을 쓰면 어느 계산이 더 바깥이든 전체 표면을 포함합니다.
5. 고정 epsilon과 `nextafter` 기반 바깥 확장의 차이.
   - **모범답변:** epsilon은 기하 경계 판정의 허용 폭을 주지만 절대 scale에 민감합니다. `nextafter`는 최종 숫자를 정확히 한 표현값만 바깥으로 옮겨 타이트함을 거의 보존하며, 원본은 둘을 보완적으로 사용합니다.

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
  - **모범답변:** 계산과 구현이 단순하고 대체로 긴 공간 방향을 줄이지만 primitive 분포와 실제 ray 비용을 반영하지 않아 겹침이 큰 트리를 만들 수 있습니다. SAH보다 build는 싸지만 traversal 품질은 낮을 수 있습니다.
- centroid가 모두 같은 경우에도 recursion이 종료되게 하려면 무엇이 필요합니까?
  - **모범답변:** centroid가 같아도 shape index로 총순서를 만들고, 기하 위치가 아니라 원소 개수의 중앙 `first + count/2`에서 나눠야 합니다. 그러면 두 범위가 비어 있지 않고 매 단계 엄격히 작아집니다.
- 연속 `nodes` 저장소와 별도 primitive-index 배열을 쓰는 이유는 무엇입니까?
  - **모범답변:** node와 leaf index를 연속 메모리에서 순회해 cache locality를 높이고, 도형 객체를 이동·복사하지 않은 채 Scene index로 참조할 수 있습니다. 구축 중 vector 재할당이 있으므로 node reference 대신 index를 보관합니다.
- 순회에서 가까운 child를 먼저 방문하면 correctness가 아니라 성능에 어떤 이점이 있습니까?
  - **모범답변:** 가까운 실제 hit를 일찍 찾으면 `closest`가 빨리 작아집니다. 이후 먼 child의 AABB나 primitive가 이 상한으로 더 많이 탈락하지만, 양쪽을 계약대로 검사한다면 최종 correctness 자체는 순서와 무관합니다.
- 현재 node entry가 이미 찾은 `closest`보다 크면 왜 prune할 수 있습니까?
  - **모범답변:** node 안의 모든 도형은 그 AABB 안에 있고 광선이 그 상자에 들어가는 가장 이른 시점이 entry입니다. entry조차 `closest`보다 멀면 subtree 안에 더 가까운 hit가 존재할 수 없습니다.
- 동일 `t`의 두 도형에서 선형 순회와 BVH 순회 순서가 다르더라도 같은 도형을 선택하려면 어떻게 해야 합니까?
  - **모범답변:** 후보 갱신을 거리만이 아니라 안정적인 shape index까지 포함한 총순서로 정의해야 합니다. 원본은 `t`가 같으면 더 큰 index를 택해 방문 순서와 무관한 결과를 만듭니다.
- unbounded 도형은 BVH 결과와 언제 합쳐야 합니까?
  - **모범답변:** BVH bounded traversal 후 현재 `closest`를 유지한 채 unbounded index를 같은 `testShape`로 검사합니다. 순서는 달라도 공통 tie 정책이 있으므로 최종 결과는 선형 모드와 동치입니다.

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
void Bvh::build(std::vector<BvhPrimitive> primitives) {
    nodes_.clear();
    primitiveIndices_.clear();
    if (primitives.empty()) return;

    // 재귀 중 node vector reference를 보관하지 않으며 index만 사용한다.
    nodes_.reserve(primitives.size() * 2);
    buildNode(primitives, 0,
              static_cast<std::uint32_t>(primitives.size()));
    primitiveIndices_.reserve(primitives.size());
    for (const BvhPrimitive& primitive : primitives)
        primitiveIndices_.push_back(primitive.shapeIndex);
}

std::uint32_t Bvh::buildNode(std::vector<BvhPrimitive>& primitives,
                             std::uint32_t first,
                             std::uint32_t last) {
    Aabb nodeBounds = primitives[first].bounds;
    Vec3 centroidMin = primitives[first].bounds.centroid();
    Vec3 centroidMax = centroidMin;
    for (std::uint32_t i = first + 1; i < last; ++i) {
        nodeBounds = surroundingBox(nodeBounds, primitives[i].bounds);
        const Vec3 c = primitives[i].bounds.centroid();
        centroidMin.x = std::min(centroidMin.x, c.x);
        centroidMin.y = std::min(centroidMin.y, c.y);
        centroidMin.z = std::min(centroidMin.z, c.z);
        centroidMax.x = std::max(centroidMax.x, c.x);
        centroidMax.y = std::max(centroidMax.y, c.y);
        centroidMax.z = std::max(centroidMax.z, c.z);
    }

    const std::uint32_t nodeIndex =
        static_cast<std::uint32_t>(nodes_.size());
    nodes_.push_back(BvhNode{});
    nodes_[nodeIndex].bounds = nodeBounds;

    constexpr std::uint32_t kMaxLeafSize = 4;
    const std::uint32_t count = last - first;
    if (count <= kMaxLeafSize) {
        nodes_[nodeIndex].first = first;
        nodes_[nodeIndex].count = count;
        return nodeIndex;
    }

    const Vec3 extent = centroidMax - centroidMin;
    int axis = 0;
    if (extent.y > extent.x) axis = 1;
    const auto component = [](const Vec3& v, int a) {
        return a == 0 ? v.x : (a == 1 ? v.y : v.z);
    };
    if (component(extent, 2) > component(extent, axis)) axis = 2;

    std::stable_sort(primitives.begin() + first, primitives.begin() + last,
        [axis, &component](const BvhPrimitive& a, const BvhPrimitive& b) {
            const double ca = component(a.bounds.centroid(), axis);
            const double cb = component(b.bounds.centroid(), axis);
            if (ca != cb) return ca < cb;
            return a.shapeIndex < b.shapeIndex; // 동일 centroid의 총순서
        });

    const std::uint32_t middle = first + count / 2;
    const std::uint32_t left = buildNode(primitives, first, middle);
    const std::uint32_t right = buildNode(primitives, middle, last);
    nodes_[nodeIndex].left = left;
    nodes_[nodeIndex].right = right;
    return nodeIndex;
}
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
   - **모범답변:** 중앙 분할은 정렬 뒤 중앙 index만 고르면 되어 구현과 결정성 검증이 쉽고 O(n log n)을 목표로 할 수 있습니다. SAH는 예상 traversal 비용을 더 잘 줄일 수 있지만 후보 분할 평가와 build 비용·복잡도가 커집니다.
2. centroid·shape index 총순서가 결정성에 미치는 영향.
   - **모범답변:** centroid만 비교하면 동률 원소의 순서가 입력이나 정렬 세부에 의존할 수 있습니다. shape index를 두 번째 key로 두면 모든 primitive의 순서가 고정되어 node와 leaf index 배열도 재현됩니다.
3. 연속 node·primitive index 표현의 cache locality와 포인터 안정성.
   - **모범답변:** 순회 중 node와 leaf index를 연속 접근해 cache에 유리합니다. 대신 build 중 vector 재할당이 주소를 바꾸므로 재귀 전후에 reference나 pointer를 유지하지 않고 node index로 다시 접근해야 합니다.
4. 최근접 순회에서 near-first와 `closest` pruning이 작동하는 방식.
   - **모범답변:** child AABB entry를 비교해 가까운 쪽을 먼저 pop하도록 stack에 먼 쪽부터 넣습니다. 실제 hit로 `closest`가 줄면 entry가 더 큰 stack 항목과 그 하위 primitive를 검사하지 않습니다.
5. 순회 순서와 무관한 equal-distance tie 계약이 필요한 이유.
   - **모범답변:** BVH build와 near-first 때문에 선형 순회와 방문 순서가 다릅니다. 동일 거리에서 shape index 같은 안정적 key 없이 현재 후보를 유지하거나 덮으면 mode별 material·normal·pixel이 달라질 수 있습니다.

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
  - **모범답변:** 시간은 사용자가 체감하는 전체 실행 비용이지만 환경 잡음도 포함합니다. counter는 같은 ray workload에서 알고리즘이 수행한 구조적 검사량을 보여 주어 BVH가 primitive 검사를 실제로 줄였는지 더 안정적으로 설명합니다.
- 가장 빠른 값이나 평균 대신 median을 쓴 이유는 무엇입니까?
  - **모범답변:** 가장 빠른 값은 유리한 우연만 선택하고 평균은 느린 outlier에 크게 끌립니다. 정렬한 반복의 중앙 sample은 일시적인 스케줄링·백그라운드 지연 영향에 덜 민감합니다.
- 선형과 BVH의 checksum이 다르면 성능 수치를 왜 폐기해야 합니까?
  - **모범답변:** 두 모드가 다른 이미지를 계산했다면 같은 문제를 더 빠르게 푼 비교가 아닙니다. 누락된 hit로 일이 줄었을 수도 있으므로 correctness 동치가 성능 주장보다 먼저입니다.
- benchmark에 자동 thread 선택을 두면 비교 해석이 어려워지는 이유는 무엇입니까?
  - **모범답변:** 머신과 실행 시점에 따라 worker 수가 달라지고 병렬 스케줄링 효과가 BVH 효과와 섞입니다. 원본 benchmark는 `threadCount=1`로 고정해 가속 구조 차이만 비교합니다.
- warm-up 한 번으로 충분하다고 일반화할 수 있습니까?
  - **모범답변:** 아닙니다. 원본 workload에서는 한 번을 사용하지만 JIT, 캐시, 동적 주파수, 파일 I/O 등 환경에 따라 더 많은 예열이나 별도 실험이 필요할 수 있습니다. 예열 횟수도 결과 설정에 기록해야 합니다.
- `primitiveTests` 감소율을 gate로 쓰고 속도 배율은 보고만 하는 설계의 장점은 무엇입니까?
  - **모범답변:** 시간 gate는 CI 부하와 하드웨어 차이로 흔들리기 쉽지만 primitive count는 동일 입력에서 결정적입니다. 원본은 BVH가 선형의 25% 미만 primitive test를 쓰는지를 구조적 gate로 삼고 speedup은 관찰값으로 남깁니다.
- reference JSON에 compiler·architecture·logical threads를 남겨야 하는 이유는 무엇입니까?
  - **모범답변:** 같은 코드도 컴파일러 최적화, ISA, 코어 구성에 따라 시간이 달라집니다. 환경 metadata가 있어야 reference 수치의 적용 범위를 판단하고 다른 환경의 값을 보편 기준으로 오해하지 않습니다.

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
Sample measure(const Scene& scene, AccelMode mode,
               int warmupRuns, int measuredRuns) {
    if (warmupRuns < 0 || measuredRuns <= 0)
        throw std::invalid_argument("invalid benchmark run count");

    for (int i = 0; i < warmupRuns; ++i) (void)render(scene, mode);

    std::vector<Sample> samples;
    samples.reserve(static_cast<std::size_t>(measuredRuns));
    for (int i = 0; i < measuredRuns; ++i)
        samples.push_back(render(scene, mode));

    std::sort(samples.begin(), samples.end(),
              [](const Sample& a, const Sample& b) {
                  return a.milliseconds < b.milliseconds;
              });
    // 원본과 같은 upper median 정책이며 stats/checksum도 같은 sample에서 온다.
    const Sample median = samples[samples.size() / 2];

    const auto sameWork = [](const RenderStats& a, const RenderStats& b) {
        return a.primaryRays == b.primaryRays &&
               a.secondaryRays == b.secondaryRays &&
               a.shadowRays == b.shadowRays &&
               a.primitiveTests == b.primitiveTests &&
               a.aabbTests == b.aabbTests;
    };
    for (const Sample& sample : samples) {
        if (sample.checksum != median.checksum ||
            !sameWork(sample.stats, median.stats)) {
            throw std::runtime_error(
                "benchmark runs produced different results");
        }
    }
    return median;
}
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
   - **모범답변:** 시간은 실제 효과를 보여 주지만 환경 잡음을 포함하고, counter는 알고리즘이 수행한 결정적 작업량을 보여 주지만 각 작업의 실제 비용은 말하지 못합니다. 둘을 함께 봐야 구조적 개선과 체감 개선을 구분할 수 있습니다.
2. median과 warm-up이 줄이는 노이즈와 줄이지 못하는 노이즈.
   - **모범답변:** warm-up은 초기 cache와 일회성 초기화 영향을 줄이고 median은 소수의 지연 outlier 영향을 줄입니다. 지속적인 thermal throttling, 다른 프로세스 부하, 머신·컴파일러 차이는 제거하지 못합니다.
3. correctness equivalence를 성능 비교보다 먼저 검사하는 원칙.
   - **모범답변:** 반복 내 checksum과 작업량이 같고, 두 모드 checksum도 같아야 동일 workload와 결과를 비교한 것입니다. 이 검사를 통과하지 못한 속도 향상은 계산 누락일 수 있어 의미가 없습니다.
4. 단일 thread로 알고리즘 효과를 분리한 판단.
   - **모범답변:** worker 수를 1로 고정하면 thread 생성, 스케줄링, load balancing 변수가 제거되어 선형 탐색과 BVH 자체의 차이를 해석하기 쉽습니다.
5. 환경·schema 기록과 자동 gate를 어디까지 신뢰할 수 있는지.
   - **모범답변:** schema와 환경은 결과 해석과 형식 변경 추적에 필요하고, 결정적 primitive 감소 gate는 CI에서도 비교적 안정적입니다. wall-clock reference는 같은 환경에서도 변동하므로 절대적 합격 기준보다는 추세와 보고 자료로 봐야 합니다.

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
