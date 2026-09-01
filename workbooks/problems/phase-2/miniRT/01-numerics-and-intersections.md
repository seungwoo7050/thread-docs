# 수치 안정성·교차·카메라 워크북

이 문서는 P01~P04를 다룬다. 백지 구현에서는 원본 구현을 보지 않고 요구사항과 invariant만으로 작성한다. 제시된 skeleton은 인터페이스 경계만 보여 주며 알고리즘 순서나 정답 코드를 포함하지 않는다.

---

<a id="p01"></a>
## [Thread 01 / `fix(math): 큰 유한 벡터를 안정적으로 정규화`] 큰 유한 벡터와 정규화 계약

### 면접 질문

`Vec3(1e308, 0, 0)`은 각 성분이 유한한데도 `sqrt(x*x + y*y + z*z)`로 길이를 계산하면 정규화가 깨질 수 있습니다. 왜 그런지 설명하고, 이 프로젝트에서 길이 계산을 어떻게 안정화했는지 말해 보세요.

꼬리 질문:

- 정규화 대상이 영벡터 또는 `kEpsilon` 이하라면 반환값과 오류 처리 중 무엇을 택하겠습니까?
- 입력에 NaN이나 무한대가 들어올 수 있다면 수학 함수와 parser 중 어느 경계에서 막아야 합니까?
- `lengthSquared()`는 여전히 overflow할 수 있는데 남겨 둘 이유와 사용 시 주의점은 무엇입니까?

### 30초 모범 답변

성분이 유한해도 제곱의 중간값은 `double` 범위를 넘을 수 있어서 길이가 무한대가 되고, 그 값으로 나누면 정상 단위벡터를 얻지 못합니다. 그래서 길이는 스케일링을 내부에서 처리하는 `std::hypot` 계열로 계산하고, 길이가 epsilon 이하인 벡터는 정규화 불가능한 값으로 명시적으로 다룹니다. 이 프로젝트에서는 외부 장면 입력의 NaN·무한대는 parser가 거부하고, 수학 계층은 큰 유한값을 안정적으로 처리하도록 역할을 나눴습니다. 영벡터를 0으로 돌려주는 정책은 호출이 단순해지지만 오류를 숨길 수 있으므로, 카메라·원기둥처럼 0이 허용되지 않는 경계에서는 별도 검증이 필요합니다.

### 답변 핵심 키워드

`중간 overflow`, `유한 입력`, `std::hypot`, `스케일링`, `near-zero`, `계층별 검증`, `영벡터 정책`, `lengthSquared 사용 주의`

### 백지 구현

**구현 목표**

큰 유한 성분에서도 overflow 때문에 잘못된 결과를 만들지 않는 벡터 길이와 정규화를 구현한다.

**인터페이스**

```cpp
namespace ray {

struct Vec3 {
    double x;
    double y;
    double z;

    double length() const;
};

Vec3 normalize(const Vec3& value);

}  // namespace ray
```

**입력과 출력**

- 입력: 세 `double` 성분을 가진 벡터.
- 출력: 길이는 음수가 아닌 `double`, 정규화는 가능한 경우 길이 1인 벡터.

**반드시 만족해야 할 조건**

- `(1e308, 0, 0)`처럼 큰 유한 입력을 정규화하면 유한한 단위벡터가 나와야 한다.
- 정상 크기 입력은 기존 벡터 연산의 의미를 유지해야 한다.
- 길이가 `kEpsilon` 이하인 입력의 처리 정책이 일관되어야 한다.
- 정규화 결과의 방향은 입력과 같아야 한다.

**경계 조건**

- 영벡터.
- `kEpsilon` 바로 아래와 위.
- 한 성분만 매우 큰 벡터.
- 크기가 크게 다른 성분이 섞인 벡터.
- 음수 성분만 있는 벡터.

**실패 조건**

- 유한 입력에서 중간 제곱 overflow로 길이가 무한대가 되는 구현.
- near-zero 입력에서 0으로 나누는 구현.
- 정규화 뒤 NaN이 생기는데도 정상 결과로 반환하는 구현.

**제약**

- 제곱합을 먼저 계산하는 방식만으로 해결하지 않는다.
- 외부 수학 라이브러리는 사용하지 않는다.
- 원본 구현과 동일한 코드 모양일 필요는 없지만 위 계약을 만족해야 한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] `(3, 4, 0)`의 길이와 정규화 결과가 기대한 값인가.
- [ ] `(1e308, 0, 0)`의 결과가 유한하고 방향이 보존되는가.
- [ ] `(1e308, 1, 0)`처럼 크기 차이가 큰 성분에서도 결과가 유한한가.
- [ ] 영벡터와 epsilon 경계에서 정한 정책이 일관되는가.
- [ ] 입력 벡터를 변경하지 않는가.
- [ ] 정상 입력에서 정규화 결과의 길이가 허용 오차 안에서 1인가.
- [ ] 시간 복잡도 O(1), 추가 공간 O(1)인가.

### 구현 후 설명할 것

1. 단순 제곱합이 유한 입력에서도 실패하는 이유.
2. 길이 계산과 입력 유효성 검증을 어느 계층에 배치했는지.
3. near-zero에서 0 반환과 예외 중 선택한 정책 및 단점.
4. `lengthSquared()`를 비교 최적화에 쓸 수 있는 범위와 위험.

### 원본 확인 위치

- Thread 01
- `fix(math): 큰 유한 벡터를 안정적으로 정규화`
- `test(math): 큰 유한 벡터 정규화 검증`
- `include/ray/math.hpp`
- `src/math.cpp`
- `tests/core_tests.cpp`
- `Vec3::length`, `normalize`, `kEpsilon`
- 관련 Thread: 03의 `requireNonzeroVector`, 04의 `buildCameraFrame`, 07의 경계 계산

---

<a id="p02"></a>
## [Thread 02 / `feat(geometry): hit와 도형 교차 계약 정의`] HitRecord와 구·평면의 유효 구간 교차

### 면접 질문

`Shape::intersect(ray, t_min, t_max, hit)`가 단순히 교차 여부만 반환하지 않고 유효한 `t` 구간과 `HitRecord::setFaceNormal` 계약을 갖는 이유를 설명해 보세요. 구 내부에서 시작한 광선과 뒷면에서 맞는 평면은 어떻게 처리해야 합니까?

꼬리 질문:

- 구의 두 root 중 가까운 root가 구간 밖이고 먼 root는 구간 안이면 어떻게 해야 합니까?
- `frontFace`와 저장된 `normal`을 분리하는 이유는 무엇입니까?
- 접선 교차, 방향이 0에 가까운 광선, 평면과 거의 평행한 광선은 어떻게 분류합니까?
- 여러 도형을 순회할 때 `t_max`를 현재 최단 거리로 줄이면 어떤 이점이 있습니까?
- equal-distance hit가 두 개일 때 정책을 명시해야 하는 이유는 무엇입니까?

### 30초 모범 답변

교차 함수의 계약은 `[t_min, t_max]` 안에서 유효한 가장 가까운 hit만 기록하는 것입니다. 이 구간이 있으면 self-intersection을 피하고 이미 찾은 거리보다 먼 후보를 버릴 수 있습니다. `setFaceNormal`은 외향 법선과 광선 방향의 내적으로 앞면 여부를 기록하고, 실제 저장 법선은 항상 입사 방향의 반대로 맞춰 셰이딩과 반사 코드가 면의 안팎을 매번 분기하지 않게 합니다. 구는 가까운 root를 먼저 시도하되 구간 밖이면 먼 root도 확인해야 내부 시작 광선을 놓치지 않습니다. 평면은 분모가 epsilon 이하일 때 평행으로 보고 거부합니다.

### 답변 핵심 키워드

`[t_min,t_max]`, `최근접 hit`, `두 root`, `내부 시작`, `frontFace`, `oriented normal`, `평행 epsilon`, `self-hit 방지`, `tie policy`

### 백지 구현

**구현 목표**

공통 hit 계약을 정의하고, 그 계약에 맞는 구 교차와 평면 교차를 구현한다.

**인터페이스**

```cpp
namespace ray {

struct HitRecord {
    double t;
    Vec3 point;
    Vec3 normal;
    bool frontFace;

    void setFaceNormal(const Ray& ray, const Vec3& outwardNormal);
};

class Sphere {
public:
    bool intersect(const Ray& ray,
                   double tMin,
                   double tMax,
                   HitRecord& hit) const;
};

class Plane {
public:
    bool intersect(const Ray& ray,
                   double tMin,
                   double tMax,
                   HitRecord& hit) const;
};

}  // namespace ray
```

**입력과 출력**

- 입력: 광선, 허용 `t` 구간, 도형 파라미터.
- 출력: 구간 내 교차가 있으면 `true`와 완성된 `HitRecord`, 없으면 `false`.

**반드시 만족해야 할 조건**

- 성공 시 `hit.t`는 허용 구간 안에 있어야 한다.
- `hit.point`는 `ray.at(hit.t)`와 일치해야 한다.
- `hit.normal`은 광선 방향과 같은 방향을 향하지 않아야 한다.
- `frontFace`는 외향 법선 기준으로 계산되어야 한다.
- 구의 두 root를 모두 고려해야 한다.
- 실패 시 호출자가 이전에 보관하던 유효 hit를 임의로 깨뜨리지 않는 방식을 택한다.

**경계 조건**

- 구 밖에서 두 점 교차.
- 구 안에서 한 방향으로 나가는 광선.
- 접선.
- 두 root가 모두 구간 밖인 경우.
- 평면과 거의 평행한 광선.
- 광선 시작점이 평면 위에 있는 경우와 `t_min`의 관계.

**실패 조건**

- 가까운 root만 보고 내부 시작 광선을 miss 처리.
- 평면 분모가 0에 가까운데 나눗셈 수행.
- normal 방향이 호출 경로마다 달라 셰이딩 결과가 뒤집힘.
- 구간 밖 hit를 성공으로 기록.

**제약**

- `ray.direction`이 항상 단위벡터라고 가정하지 않는다.
- 비교 epsilon과 ray 시작 offset의 역할을 혼동하지 않는다.
- 원본 코드의 helper 구조를 복사할 필요는 없다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 정면에서 구를 맞히면 가까운 root가 선택되는가.
- [ ] 구 내부 시작 광선은 먼 root로 빠져나오는 hit를 찾는가.
- [ ] 접선이 프로젝트에서 정한 정책대로 hit로 처리되는가.
- [ ] `t_min`보다 작은 교차는 제외되는가.
- [ ] 평면 앞·뒤 어느 쪽에서 접근해도 normal이 입사 반대 방향인가.
- [ ] 거의 평행한 광선에서 NaN이나 매우 큰 잘못된 `t`가 나오지 않는가.
- [ ] aggregate 순회에서 현재 최단 `t`를 새 `t_max`로 전달해도 계약이 유지되는가.
- [ ] 각 함수가 O(1) 시간과 O(1) 추가 공간을 사용하는가.

### 구현 후 설명할 것

1. outward normal, `frontFace`, 저장 normal을 각각 왜 구분했는지.
2. `t_min`과 `t_max`가 correctness와 성능에 미치는 영향.
3. 구 내부 시작·접선·평면 평행을 어떻게 판정했는지.
4. equal-distance 후보의 정책을 aggregate 계층에서 정해야 하는 이유.
5. 실패 시 output parameter를 언제 갱신하는지.

### 원본 확인 위치

- Thread 02
- `feat(geometry): hit와 도형 교차 계약 정의`
- `feat(geometry): 구 교차 계산 구현`
- `feat(geometry): 평면 교차 계산 구현`
- `include/ray/geometry.hpp`
- `src/geometry.cpp`
- `HitRecord`, `HitRecord::setFaceNormal`, `Shape::intersect`, `Sphere::intersect`, `Plane::intersect`
- 관련 Thread: 05의 `Scene::intersect`·`findNearestHit`, 08의 linear/BVH equal-distance 계약

---

<a id="p03"></a>
## [Thread 02 / `feat(geometry): 원기둥 cap과 최근접 hit 선택 완성`] 유한 원기둥의 side·cap 후보 통합

### 면접 질문

임의 방향 축을 가진 유한 원기둥과 광선의 교차를 어떻게 나눠 계산했는지 설명해 보세요. 무한 원기둥의 옆면 root만 구하면 왜 충분하지 않으며, cap과 옆면이 모두 hit일 때 어떤 계약으로 하나를 선택해야 합니까?

꼬리 질문:

- 광선 방향이 원기둥 축과 평행하면 옆면 방정식의 어떤 항이 퇴화합니까?
- 옆면 후보가 원기둥 높이 범위 안인지 어떻게 판정합니까?
- cap 평면 교차점이 원 안에 있는지 어떤 값으로 검사합니까?
- cap의 rim처럼 side와 cap이 같은 거리에 잡힐 수 있는 경우 정책은 무엇입니까?
- `update_hit_if_closer` 같은 helper로 후보 갱신을 통합하면 어떤 오류를 줄일 수 있습니까?

### 30초 모범 답변

광선 방향과 원점 오프셋을 원기둥 축 방향 성분과 수직 성분으로 분해하면 옆면은 축에 수직한 2차식으로 계산할 수 있습니다. 얻은 root마다 hit 지점의 축 방향 거리가 반높이 범위 안인지 확인해야 무한 원기둥을 유한하게 자를 수 있습니다. 축과 거의 평행하면 옆면 2차식이 퇴화하므로 건너뛰고 cap은 별도의 평면 교차 후 중심에서의 반지름 검사로 처리합니다. side와 두 cap이 모두 후보가 될 수 있으므로 모든 성공 후보를 같은 `[t_min, closest]` 계약으로 갱신해 가장 가까운 hit 하나만 남깁니다.

### 답변 핵심 키워드

`축/수직 성분 분해`, `무한 원기둥`, `half height`, `퇴화한 quadratic`, `cap plane`, `disk test`, `후보 통합`, `closest`, `rim`

### 백지 구현

**구현 목표**

정규화된 임의 축을 가진 닫힌 유한 원기둥의 옆면과 두 cap을 모두 고려해 가장 가까운 hit를 반환한다.

**인터페이스**

```cpp
namespace ray {

class Cylinder {
public:
    bool intersect(const Ray& ray,
                   double tMin,
                   double tMax,
                   HitRecord& hit) const;
};

}  // namespace ray
```

**입력과 출력**

- 입력: 원기둥 중심, 정규화된 축, 반지름, 높이, 광선, 유효 `t` 구간.
- 출력: side 또는 cap 중 구간 내 가장 가까운 hit.

**반드시 만족해야 할 조건**

- 반지름·높이가 유효하지 않거나 축이 퇴화하면 hit가 없어야 한다.
- 옆면 root는 높이 범위 안에서만 유효하다.
- 두 cap의 유효 원판 내부만 hit로 인정한다.
- 여러 후보 중 가장 가까운 후보만 `HitRecord`에 남는다.
- 법선은 P02의 `setFaceNormal` 계약을 따른다.

**경계 조건**

- 축과 평행한 광선.
- 원기둥 내부에서 옆면으로 나가는 광선.
- cap 중심을 수직으로 통과하는 광선.
- cap 평면은 맞지만 원판 밖인 광선.
- side와 cap의 경계인 rim.
- 접선과 discriminant 0 부근.
- `t_min` 바로 앞의 self-hit 후보.

**실패 조건**

- 무한 원기둥 root를 높이 검사 없이 반환.
- 퇴화한 2차식에서 0으로 나눔.
- cap의 평면 교차만 확인하고 반지름 검사를 생략.
- 먼 후보가 가까운 후보를 덮어씀.
- side와 cap이 서로 다른 normal 방향 계약을 사용.

**제약**

- 원기둥 축은 임의 방향일 수 있다.
- 회전 행렬로 로컬 좌표계에 옮기는 해법과 성분 분해 해법 모두 허용한다.
- 구현 시간은 30분 안으로 제한하고, 재질 복사·type name 같은 주변 코드는 생략한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 정면 옆면 hit의 거리와 normal이 맞는가.
- [ ] 축 방향 광선이 cap을 맞히고 옆면 계산에서 오류가 나지 않는가.
- [ ] 높이 밖의 무한 원기둥 root를 거부하는가.
- [ ] cap 평면의 원판 밖 점을 거부하는가.
- [ ] 원기둥 내부 시작 광선이 올바른 출구 hit를 찾는가.
- [ ] rim과 접선에서 정책이 일관되고 NaN이 없는가.
- [ ] side·top cap·bottom cap 중 최소 `t`가 선택되는가.
- [ ] 성공하지 않은 후보가 기존 `hit`를 오염시키지 않는가.
- [ ] 시간·공간 복잡도가 O(1)인가.

### 구현 후 설명할 것

1. 축 성분 분해 또는 로컬 좌표 변환 중 선택한 방법과 이유.
2. side·cap 공통 후보 갱신 로직을 어떻게 일관되게 만들었는지.
3. epsilon을 discriminant, 높이, rim 판정에 어떻게 적용했는지.
4. 축이 정규화되어 있다는 invariant를 생성자와 `intersect` 중 어디서 보장했는지.
5. 열린 원기둥으로 바꾸면 어떤 부분을 제거하면 되는지.

### 원본 확인 위치

- Thread 02
- `feat(geometry): 유한 원기둥 옆면 교차 구현`
- `feat(geometry): 원기둥 cap과 최근접 hit 선택 완성`
- `include/ray/geometry.hpp`
- `src/geometry.cpp`
- `Cylinder::intersect`, `update_hit_if_closer`, `test_cylinder_cap`
- 관련 Thread: 03의 `feat(parser): 원기둥 지시어 지원`, 07의 `Cylinder::bounds`

---

<a id="p04"></a>
## [Thread 04 / `feat(camera): 화면 좌표를 카메라 광선으로 변환`] 직교 카메라 프레임과 프레임 재사용

### 면접 질문

카메라의 position, direction, up, FOV와 이미지 크기로부터 픽셀 중심을 지나는 광선을 만드는 과정을 설명해 보세요. direction과 up이 거의 평행하면 어떤 문제가 생기며, 픽셀마다 프레임을 다시 만들지 않도록 최적화할 때 무엇을 검증해야 합니까?

꼬리 질문:

- `right = cross(up, forward)`와 반대 순서는 화면 좌우에 어떤 영향을 줍니까?
- 세로 좌표에서 `0.5 - y/height` 형태가 필요한 이유는 무엇입니까?
- FOV를 degree에서 radian으로 바꾸고 `tan(fov/2)`를 쓰는 이유는 무엇입니까?
- width 또는 height가 0인 입력을 카메라 계층에서 보정할지, 상위 계층에서 거부할지 어떻게 결정합니까?
- cached frame과 uncached path의 결과를 비트 단위로 같게 만들려면 계산 순서가 왜 중요합니까?

### 30초 모범 답변

먼저 direction을 forward로 정규화하고 up과 거의 평행하면 다른 기준 축을 선택합니다. 그다음 cross product로 right와 실제 up을 만들어 직교 기저를 구성합니다. 세로 FOV의 절반 각도에 대한 tangent로 viewport 높이를 구하고 aspect ratio로 폭을 계산한 뒤, 픽셀 중심을 정규화된 화면 좌표로 옮겨 `forward + right*u + up*v`를 정규화합니다. 프레임은 모든 픽셀에 공통이므로 한 번만 계산해 재사용하되, 기존 경로와 연산 순서가 달라져 결과가 변하지 않는지 동일 광선을 비교해야 합니다.

### 답변 핵심 키워드

`orthonormal basis`, `forward/right/up`, `평행 up fallback`, `cross 순서`, `aspect ratio`, `tan(FOV/2)`, `pixel center`, `Y flip`, `frame cache`, `동치`

### 백지 구현

**구현 목표**

퇴화한 up 입력을 안전하게 처리하는 카메라 프레임을 만들고, 주어진 픽셀 중심에 대한 정규화된 광선을 생성한다.

**인터페이스**

```cpp
namespace ray {

struct CameraFrame {
    Vec3 forward;
    Vec3 right;
    Vec3 up;
    double viewportWidth;
    double viewportHeight;
};

CameraFrame buildCameraFrame(const Camera& camera,
                             int width,
                             int height);

Ray makeCameraRay(const Camera& camera,
                  const CameraFrame& frame,
                  int width,
                  int height,
                  double pixelX,
                  double pixelY);

}  // namespace ray
```

**입력과 출력**

- 입력: 카메라, 이미지 크기, 연속 좌표의 픽셀 중심.
- 출력: 카메라 위치에서 시작하는 정규화된 방향의 `Ray`.

**반드시 만족해야 할 조건**

- `forward`, `right`, `up`은 유효한 직교 기저여야 한다.
- direction과 up이 거의 평행해도 NaN이 생기지 않아야 한다.
- 중앙 픽셀의 방향은 forward와 일관되어야 한다.
- 이미지 aspect ratio가 viewport 폭에 반영되어야 한다.
- cached frame을 사용한 경로와 매번 frame을 만든 경로가 같은 의미를 가져야 한다.

**경계 조건**

- direction이 near-zero인 카메라.
- up이 near-zero인 카메라.
- up과 direction이 같은 방향 또는 반대 방향.
- 세로로 긴 이미지와 가로로 긴 이미지.
- 이미지 모서리 픽셀.
- 매우 작은 또는 180도에 가까운 FOV는 parser 계약과 연결해 처리한다.

**실패 조건**

- cross product 결과가 near-zero인데 정규화.
- cross 순서가 뒤집혀 좌우 또는 상하가 반전.
- degree 값을 그대로 `tan`에 전달.
- 픽셀 모서리 좌표를 사용해 반 픽셀 오차 발생.
- 매 픽셀 공통 프레임을 반복 계산.

**제약**

- parser가 유효 FOV를 보장한다고 가정할 수 있으나, 함수의 자체 방어 범위를 설명해야 한다.
- 행렬 라이브러리는 사용하지 않는다.
- 원본의 상수나 문장 구조를 복사하지 않는다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 중앙 광선이 forward와 같은 방향인가.
- [ ] 좌우 픽셀의 방향 변화가 `right` 축과 일관되는가.
- [ ] 화면 위쪽 픽셀의 방향 변화가 선택한 Y convention과 일치하는가.
- [ ] 기저 벡터들의 길이가 1이고 상호 내적이 허용 오차 안에서 0인가.
- [ ] up과 forward가 평행한 입력에서도 유한한 프레임이 생성되는가.
- [ ] 16:9와 9:16에서 viewport 비율이 올바른가.
- [ ] cached/uncached 경로의 동일 입력 결과가 같은가.
- [ ] 프레임 구축은 프레임당 O(1), 픽셀 광선 생성은 픽셀당 O(1)인가.

### 구현 후 설명할 것

1. 좌표계 handedness와 cross product 순서를 어떻게 정했는지.
2. 평행 up fallback 축을 선택하는 기준.
3. FOV와 viewport 크기의 관계.
4. frame cache가 안전한 이유와 카메라 변경 시 invalidation 필요성.
5. 함수 내부 보정과 parser 선검증의 책임 분리.

### 원본 확인 위치

- Thread 04
- `feat(camera): 화면 좌표를 카메라 광선으로 변환`
- `perf(camera): 픽셀별 카메라 프레임 재계산 제거`
- `test(camera): 재사용한 카메라 프레임의 동치 검증`
- `include/ray/camera.hpp`
- `src/camera.cpp`
- `src/renderer.cpp`
- `CameraFrame`, `buildCameraFrame`, `makeCameraRay`
- 관련 Thread: 01의 벡터 정규화, 03의 카메라 방향·FOV 검증, 09의 프레임 단위 재사용
