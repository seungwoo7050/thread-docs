# 파서·장면 상태·셰이딩 워크북

이 문서는 P05~P08을 다룬다. 입력을 값으로 바꾸는 경계, 장면과 가속 구조의 lifecycle, 직접광·그림자, 재질 dispatch와 반사 깊이를 하나의 흐름으로 묶었다. 백지 구현은 면접 시간에 맞게 축소했으며 원본 구현 순서나 정답 코드는 제공하지 않는다.

---

<a id="p05"></a>
## [Thread 03 / `feat(parser): 유한 수와 범위 값 해석 구현`] 엄격한 장면 문법과 진단 가능한 입력 검증

### 면접 질문

장면 파일에서 숫자를 읽을 때 `std::stod`나 `std::stoll`이 값을 하나 반환했다는 사실만으로 입력이 유효하다고 볼 수 없는 이유를 설명해 보세요. 이 프로젝트의 parser는 어떤 단계에서 유한성, token 전체 소비, 값 범위, 중복 지시어, 필수 지시어를 검증했습니까?

꼬리 질문:

- `1.0abc`, `nan`, `inf`, 빈 token을 각각 어떻게 처리해야 합니까?
- 비율, 양수 길이, FOV, RGB처럼 의미가 다른 숫자에 하나의 범용 변환 함수만 쓰면 어떤 문제가 생깁니까?
- 카메라 방향이나 원기둥 축은 성분별 near-zero가 아니라 벡터 길이로 검사해야 하는 이유가 무엇입니까?
- 오류 메시지에 source name과 line number를 보존하면 테스트와 사용자 경험에 어떤 이점이 있습니까?
- 파싱 중간에 일부 `Scene` 상태가 만들어진 뒤 오류가 나면 호출자에게 무엇이 노출되어야 합니까?

### 30초 모범 답변

숫자 변환 함수는 접두부만 성공해도 값을 줄 수 있고 NaN·무한대도 표현할 수 있으므로, 변환 성공뿐 아니라 token을 끝까지 소비했는지와 `std::isfinite`까지 확인해야 합니다. 그다음 비율, 색상, FOV, 양수 크기처럼 도메인별 helper에서 범위를 검증하고, 지시어 계층에서 인자 수·중복·필수 항목을 검사합니다. 방향 벡터는 전체 길이가 epsilon 이하인지 확인해야 축마다 작은 값이 모인 퇴화 벡터도 일관되게 거부할 수 있습니다. 오류는 source와 line을 가진 `ParseError`로 올리고, 완전히 유효한 장면만 반환해 부분 상태가 외부에 노출되지 않게 합니다.

### 답변 핵심 키워드

`전체 token 소비`, `std::isfinite`, `도메인별 range`, `인자 수`, `중복 지시어`, `필수 지시어`, `벡터 길이 epsilon`, `source:line`, `부분 상태 비노출`

### 백지 구현

**구현 목표**

문자열 token을 엄격하게 숫자·비율·3차원 벡터로 변환하고, 실패 시 source와 line을 포함한 예외를 발생시키는 parser helper를 구현한다.

**인터페이스**

```cpp
class ParseError : public std::runtime_error {
public:
    ParseError(std::string source,
               std::size_t line,
               std::string message);

    const std::string& source() const noexcept;
    std::size_t line() const noexcept;
};

double parseDoubleToken(const std::string& token,
                        const std::string& source,
                        std::size_t line,
                        const std::string& field);

double parseRatio(const std::string& token,
                  const std::string& source,
                  std::size_t line,
                  const std::string& field);

Vec3 parseVec3(const std::string& token,
               const std::string& source,
               std::size_t line,
               const std::string& field);
```

**입력과 출력**

- 입력: 단일 숫자 token 또는 쉼표로 구분한 벡터 token, source 이름, line 번호, field 이름.
- 출력: 검증된 `double`, `[0, 1]` 비율, 유한한 `Vec3`.
- 실패: `ParseError`를 던지고 부분 결과를 반환하지 않는다.

**반드시 만족해야 할 조건**

- 숫자 token의 모든 문자가 하나의 숫자 표현으로 소비되어야 한다.
- 반환되는 모든 실수는 유한해야 한다.
- 비율은 닫힌 구간 `[0, 1]` 안에 있어야 한다.
- 벡터는 정확히 세 성분이어야 하며 각 성분은 같은 엄격한 숫자 계약을 따라야 한다.
- 오류에는 source, line, field가 식별 가능하게 포함되어야 한다.
- 변환 라이브러리의 예외 종류가 parser 외부 계약으로 새어 나가지 않아야 한다.

**경계 조건**

- `0`, `-0`, `1`, 소수, 지수 표기.
- `1.0abc`, 앞뒤 공백이 포함된 token, 빈 token.
- `nan`, `inf`, overflow를 유발하는 지수.
- `0`과 `1` 비율, 경계 바로 밖의 값.
- `1,2,3`, `1,2`, `1,2,3,4`, 빈 성분이 있는 벡터.

**실패 조건**

- 접두부만 변환하고 뒤 문자를 무시한다.
- NaN이나 무한대를 정상 값으로 통과시킨다.
- 범위 오류를 일반 숫자 변환 실패와 구분하지 못해 진단이 모호해진다.
- 원래 source와 line 정보를 잃는다.

**제약**

- 정규식 전체 문법을 만들 필요는 없다.
- 장면 전체 지시어 dispatch는 구현하지 않는다.
- 원본 helper 이름과 내부 구조를 그대로 따를 필요는 없지만 계약은 유지한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 정상 정수·소수·지수 표기가 모두 처리되는가.
- [ ] trailing 문자와 빈 token이 거부되는가.
- [ ] NaN·무한대·범위 초과가 거부되는가.
- [ ] 비율의 0과 1은 허용되고 그 밖은 거부되는가.
- [ ] 벡터의 성분 수가 정확히 세 개인지 검사하는가.
- [ ] 벡터 성분 하나가 실패해도 부분 `Vec3`가 노출되지 않는가.
- [ ] 모든 실패에서 source, line, field를 찾을 수 있는가.
- [ ] 예외가 난 뒤 호출자의 장면 상태가 변경되지 않도록 helper가 값 반환형으로 구성되었는가.
- [ ] 각 token 처리는 token 길이에 선형이고 불필요한 전역 상태가 없는가.

### 구현 후 설명할 것

1. 변환 성공, 전체 소비, 유한성, 도메인 범위를 서로 다른 검증 단계로 둔 이유.
2. `ParseError`가 변환 라이브러리 예외를 감싸는 장점.
3. 성분별 작은 값 검사보다 벡터 길이 검사가 적절한 경우.
4. 지시어 인자 수·중복·필수 항목 검증을 어느 계층에 둘지.
5. parser가 완성된 값만 장면에 반영하도록 만든 방법.

### 원본 확인 위치

- Thread 03
- `feat(parser): 유한 수와 범위 값 해석 구현`
- `feat(parser): 벡터와 색상 token 해석 구현`
- `feat(parser): 카메라와 광원 지시어 지원`
- `feat(parser): 원기둥 지시어 지원`
- `feat(parser): 필수 지시어 검증과 입력 loader 완성`
- `fix(parser): 임계값 이하 방향 벡터 거부`
- `test(parser): 퇴화한 카메라와 원기둥 방향 검증`
- `include/ray/parser.hpp`
- `src/parser.cpp`
- `tests/core_tests.cpp`
- `ParseError`, `parseDoubleToken`, `parseIntToken`, `parseRatio`, `parsePositiveDouble`, `parseVec3`, `parseColor`, `expectCount`, `rejectDuplicate`, `requireNonzeroVector`, `parseScene`, `parseSceneText`, `parseSceneFile`, `loadScene`
- 관련 Thread: 01의 수치 안정성, 06의 선택적 재질 문법, 08의 parse 완료 후 가속 구조 구축, 12의 CLI 입력 검증

---

<a id="p06"></a>
## [Thread 08 / `feat(scene): 가속 구조 소유권과 rebuild 경계 구성`] Scene 소유권과 BVH lifecycle invariant

### 면접 질문

`Scene`이 도형을 단독 소유하고, `addShape`가 호출될 때 BVH를 무효화하며, 다시 구축되기 전 BVH 요청은 현재 도형에 대한 선형 탐색으로 fallback하도록 한 이유를 설명해 보세요. 단순히 "도형을 추가한 뒤 `buildAcceleration()`을 호출해야 한다"는 사용 규칙만 문서로 남기는 방식과 무엇이 다릅니까?

꼬리 질문:

- BVH가 도형 객체 대신 장면의 도형 index를 저장하면 어떤 lifetime 조건이 생깁니까?
- `std::unique_ptr<Shape>`가 이 장면 모델에 적합한 이유는 무엇입니까?
- 유한 bounds가 없는 평면은 BVH 구축과 탐색에서 어떻게 다뤄야 합니까?
- 무효화된 BVH를 그대로 순회하는 것과 선형 fallback 중 correctness 차이는 무엇입니까?
- 도형의 기하 필드를 외부에서 직접 수정할 수 있다면 `addShape` 기반 무효화만으로 충분합니까?
- rebuild 실패 시 이전 BVH를 유지할지, 비활성화할지 어떤 정책을 택하겠습니까?

### 30초 모범 답변

BVH는 장면 도형의 위치와 bounds를 snapshot처럼 index로 보관하므로, 도형 집합이 바뀌면 즉시 stale 상태가 됩니다. 그래서 모든 구조 변경을 `Scene` 경계로 모으고 `addShape`가 BVH와 unbounded index를 지운 뒤 `accelerationReady=false`로 만드는 invariant를 둡니다. 준비되지 않은 상태에서 BVH 모드를 요청해도 선형 탐색으로 현재 도형을 검사하면 성능만 떨어지고 결과는 보존됩니다. `unique_ptr`는 도형의 단일 lifetime 주체가 Scene임을 분명히 하고, 외부 직접 변경은 accessor와 불변 기하 필드로 막아 무효화 경계를 우회하지 못하게 합니다. 평면처럼 bounds가 없는 도형은 별도 목록으로 항상 검사합니다.

### 답변 핵심 키워드

`단독 소유권`, `index lifetime`, `stale acceleration`, `mutation boundary`, `invalidation`, `accelerationReady`, `linear fallback`, `unbounded primitive`, `read-only geometry`, `correctness before performance`

### 백지 구현

**구현 목표**

도형을 단독 소유하고, 도형 추가 시 가속 구조를 무효화하며, rebuild 전에도 현재 장면 기준의 올바른 교차 결과를 반환하는 축소된 `Scene` lifecycle을 구현한다. `Shape::bounds`, `Bvh::build`, 실제 교차 수학은 제공된 것으로 가정한다.

**인터페이스**

```cpp
enum class AccelMode {
    Linear,
    Bvh
};

class Scene {
public:
    void addShape(std::unique_ptr<Shape> shape);
    void buildAcceleration();
    bool accelerationReady() const;

    std::size_t shapeCount() const;
    const Shape& shapeAt(std::size_t index) const;

    bool intersect(const Ray& ray,
                   double tMin,
                   double tMax,
                   HitRecord& hit,
                   AccelMode mode) const;

private:
    std::vector<std::unique_ptr<Shape>> shapes_;
    Bvh bvh_;
    std::vector<std::uint32_t> unboundedIndices_;
    bool accelerationReady_ = false;
};
```

**입력과 출력**

- 입력: 소유권이 이전되는 도형, rebuild 요청, 광선과 가속 모드.
- 출력: 장면 상태 플래그와 현재 도형 집합을 기준으로 한 최근접 hit.

**반드시 만족해야 할 조건**

- `Scene`만 각 `Shape`의 lifetime을 소유한다.
- null 도형이 전달되었을 때 정책이 명확해야 한다.
- 도형 집합이 변경되면 기존 BVH와 unbounded index는 사용할 수 없는 상태가 된다.
- BVH가 준비되지 않았으면 BVH 모드 요청도 현재 `shapes_`에 대한 선형 탐색으로 동작한다.
- rebuild는 bounds가 유효한 도형과 unbounded 도형을 분리한다.
- rebuild가 성공한 뒤에만 `accelerationReady_`가 참이 된다.
- 선형 모드와 BVH 모드는 같은 tie 정책과 hit 계약을 공유해야 한다.

**경계 조건**

- 빈 장면의 rebuild와 교차.
- bounds가 있는 도형만 있는 장면.
- 평면처럼 unbounded 도형만 있는 장면.
- bounded와 unbounded가 섞인 장면.
- rebuild 직후 새 도형을 추가하고 바로 BVH 모드로 교차.
- 동일 거리의 도형이 여러 개인 경우.

**실패 조건**

- 도형 추가 후 stale BVH를 계속 순회한다.
- unbounded 도형을 BVH에서 제외한 뒤 별도로 검사하지 않는다.
- rebuild 중간에 ready 플래그를 먼저 켠다.
- 외부가 도형 저장소나 기하 필드를 직접 수정해 무효화를 우회한다.
- 선형 fallback과 BVH가 서로 다른 tie 결과를 반환한다.

**제약**

- BVH 내부 구축 알고리즘은 구현 범위가 아니다.
- thread-safe mutation까지 요구하지 않는다.
- 도형 삭제·이동 API는 만들지 않는다. 추가라는 단일 mutation으로 invariant를 증명한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 빈 장면에서 rebuild가 안전하며 ready 상태가 일관되는가.
- [ ] 도형 추가 전후 `shapeCount`와 소유권 이전이 올바른가.
- [ ] rebuild 뒤 `accelerationReady()`가 참인가.
- [ ] rebuild 뒤 새 도형을 추가하면 즉시 거짓이 되는가.
- [ ] 무효 상태의 BVH 모드가 새로 추가한 도형까지 선형으로 찾는가.
- [ ] 다시 rebuild한 뒤 BVH가 현재 도형을 찾는가.
- [ ] unbounded 도형이 선형·BVH 양쪽에서 누락되지 않는가.
- [ ] 동일 거리 tie 정책이 두 모드에서 같은가.
- [ ] null 입력이나 out-of-range `shapeAt`의 정책이 명확한가.
- [ ] 객체가 파괴될 때 도형들이 한 번만 해제되는가.

### 구현 후 설명할 것

1. `unique_ptr`와 index 기반 BVH의 lifetime 관계.
2. mutation 시 rebuild를 즉시 하는 방식과 lazy invalidation·fallback의 trade-off.
3. stale BVH를 절대 쓰지 않게 만드는 상태 invariant.
4. unbounded 도형을 별도 경로로 관리하는 이유.
5. 외부 직접 기하 변경을 막아야 이 lifecycle이 완전해지는 이유.

### 원본 확인 위치

- Thread 08
- `refactor(scene): 장면 도형의 단독 소유권 적용`
- `feat(scene): 가속 구조 소유권과 rebuild 경계 구성`
- `include/ray/scene.hpp`
- `src/scene.cpp`
- `include/ray/geometry.hpp`
- `src/geometry.cpp`
- `tests/accel_tests.cpp`
- `Scene::addShape`, `Scene::buildAcceleration`, `Scene::accelerationReady`, `Scene::intersect`, `Scene::shapeCount`, `Scene::shapeAt`
- 관련 Thread: 03의 parse 완료 후 `buildAcceleration`, 07의 `Shape::bounds`, 08의 BVH 구축·순회, 09의 읽기 전용 장면 병렬 렌더링
- 프로젝트 기록에서 후속 구현으로 도형 저장소와 기하 필드의 외부 직접 변경을 막고 mutation boundary를 검증한 사실은 확인되지만, 해당 변경의 커밋 메시지는 노출된 기록에서 확인되지 않아 여기에는 별도 커밋명으로 적지 않았다.

---

<a id="p07"></a>
## [Thread 05 / `feat(render): 직접광과 그림자 추적 구현`] 직접광, 유한 shadow ray, self-intersection 방지

### 면접 질문

표면점에서 점광원으로 shadow ray를 쏠 때 단순히 "그 방향으로 어떤 교차가 있으면 그림자"라고 구현하면 왜 잘못됩니까? 이 프로젝트의 `isOccluded`와 `shadeHit`가 가져야 하는 거리 구간과 원점 보정 계약을 설명해 보세요.

꼬리 질문:

- 광원 뒤에 있는 물체가 그림자를 만들지 않게 하려면 `t_max`를 어떻게 잡아야 합니까?
- 표면점 그대로 광선을 시작하면 self-shadow가 생기는 이유는 무엇입니까?
- normal 방향 offset과 광선 방향 offset은 어떤 상황에서 차이가 납니까?
- `dot(normal, lightDirection) <= 0`일 때 shadow ray를 생략할 수 있는 이유는 무엇입니까?
- 광원과 hit point가 거의 같은 위치라면 어떤 처리가 필요합니까?
- 하나의 전역 epsilon이 모든 장면 scale에서 충분하지 않을 수 있는 이유는 무엇입니까?

### 30초 모범 답변

Shadow ray는 무한 광선이 아니라 표면에서 광원 직전까지의 유한 선분입니다. 따라서 `t_min`에는 self-hit을 피할 작은 양수를 쓰고, `t_max`는 광원 거리보다 조금 작게 제한해야 광원 뒤 물체를 occluder로 오인하지 않습니다. 시작점도 hit point에서 oriented normal 방향으로 조금 이동해 부동소수점 오차로 같은 표면을 다시 맞히는 문제를 줄입니다. Lambert 항이 0 이하인 빛이나 거리가 epsilon 이하인 빛은 기여가 없거나 방향을 정의할 수 없으므로 교차 검사 전에 건너뜁니다. 이 방식은 단순하고 결정적이지만 epsilon은 장면 scale에 민감하다는 trade-off가 있습니다.

### 답변 핵심 키워드

`유한 선분`, `t_min`, `distance-to-light`, `t_max 축소`, `shadow acne`, `normal offset`, `Lambert max(0,dot)`, `zero-distance light`, `scale-sensitive epsilon`

### 백지 구현

**구현 목표**

점광원 하나에 대한 diffuse 기여와 그림자 여부를 계산한다. 장면의 최근접 교차 함수는 제공된다고 가정한다.

**인터페이스**

```cpp
bool isOccluded(const Scene& scene,
                const Ray& shadowRay,
                double maxDistance,
                AccelMode mode);

Color shadeDiffuseLight(const Scene& scene,
                        const HitRecord& hit,
                        const Light& light,
                        AccelMode mode);
```

**입력과 출력**

- 입력: hit 정보, 점광원, 장면, 가속 모드.
- 출력: 광원까지 막혔는지 여부와 해당 광원의 diffuse 색상 기여.

**반드시 만족해야 할 조건**

- shadow 교차는 표면 이후부터 광원 이전까지만 검사한다.
- 광원 뒤 물체는 occlusion으로 세지 않는다.
- shadow ray 원점은 자기 표면을 다시 맞힐 가능성을 줄이는 방향으로 보정한다.
- 뒷면 광원은 0 기여를 반환한다.
- 광원까지 거리가 너무 작아 방향을 만들 수 없으면 안전하게 처리한다.
- 최종 기여는 material albedo, light color, brightness, Lambert 항을 반영한다.

**경계 조건**

- occluder가 hit와 광원 사이에 있음.
- occluder가 광원 뒤에 있음.
- 광원이 표면 normal의 반대편에 있음.
- 광원과 hit point의 거리가 epsilon 이하.
- 자기 표면이 부동소수점 오차로 `t≈0`에서 다시 맞는 경우.
- occluder가 광원 위치와 거의 같은 거리에 있음.

**실패 조건**

- `t_max`를 무한대로 두어 광원 뒤 물체가 그림자를 만든다.
- hit point를 그대로 원점으로 사용해 self-shadow가 발생한다.
- 정규화 전에 0에 가까운 거리를 나눈다.
- Lambert 음수 값을 그대로 색상에 더한다.

**제약**

- 다중 광원 누적, ambient, 최종 clamp는 구현 범위에서 제외해도 된다.
- 투명 재질과 soft shadow는 다루지 않는다.
- epsilon 선택은 상수 하나로 두되 한계를 설명할 수 있어야 한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 막는 물체가 없으면 정면 광원이 양의 기여를 주는가.
- [ ] hit와 광원 사이의 물체가 기여를 0으로 만드는가.
- [ ] 광원 뒤 물체는 결과에 영향을 주지 않는가.
- [ ] 뒷면 광원은 교차 검사를 하지 않아도 0인가.
- [ ] 광원과 점이 거의 같을 때 NaN이 생기지 않는가.
- [ ] 원점 보정으로 자기 교차가 줄어드는가.
- [ ] shadow `t` 구간의 양 끝 정책이 일관되는가.
- [ ] 한 광원당 교차 비용 외의 작업은 O(1)인가.

### 구현 후 설명할 것

1. shadow ray를 무한 광선이 아니라 유한 선분으로 본 이유.
2. 원점 offset 방향과 `t_min`을 함께 사용하는 이유.
3. 광원 기여를 계산하기 전에 early-out하는 조건들.
4. 고정 epsilon의 단순성과 scale 민감성 trade-off.
5. 가속 모드와 무관하게 같은 occlusion 계약을 유지하는 방법.

### 원본 확인 위치

- Thread 05
- `feat(render): 직접광과 그림자 추적 구현`
- `include/ray/renderer.hpp`
- `src/shading.cpp`
- `src/scene.cpp`
- `findNearestHit`, `isOccluded`, `shadeHit`, `traceRay`, `Scene::intersect`, `kRayTMin`, `kEpsilon`
- 관련 Thread: 02의 hit 구간과 oriented normal, 06의 재질 dispatch, 08의 선형·BVH 교차 동치, 10의 shadow ray 계측

---

<a id="p08"></a>
## [Thread 06 / `feat(material): metal 모델과 깊이 제한 반사 구현`] 재질 dispatch와 종료가 보장된 재귀 광선

### 면접 질문

Diffuse와 Metal을 `MaterialType`으로 dispatch하고 Metal 경로에서 `maxDepth`를 감소시키도록 한 이유를 설명해 보세요. 깊이가 0인 금속 표면, miss한 반사광선, 반사광선의 원점, albedo 적용은 각각 어떤 계약을 가져야 합니까?

꼬리 질문:

- 완전 반사 방향을 계산할 때 입력 방향과 normal이 단위벡터라는 가정은 어디에서 보장됩니까?
- `maxDepth`를 "남은 반사 횟수"와 "현재 깊이" 중 어느 의미로 정의했습니까?
- 깊이 소진 시 검정, local shading, background 중 무엇을 반환할지 정책이 중요한 이유는 무엇입니까?
- 반사 광선도 BVH/선형 모드와 통계를 그대로 전달해야 하는 이유는 무엇입니까?
- parser에서 재질 생략 시 diffuse를 기본으로 하고 알 수 없는 이름을 거부한 이유는 무엇입니까?
- enum dispatch와 가상 재질 객체 방식의 trade-off는 무엇입니까?

### 30초 모범 답변

재질별 광선 흐름이 다르므로 hit 뒤에 `MaterialType`으로 명시적으로 dispatch했습니다. Diffuse는 직접광을 계산하고, Metal은 `d - 2·dot(d,n)·n` 방향으로 새 광선을 만들되 normal 쪽으로 원점을 보정하고 남은 깊이를 하나 줄여 재귀합니다. `maxDepth`는 남은 반사 횟수라서 0인 Metal 경로는 더 이상 secondary ray를 만들지 않고 프로젝트 계약상 검정을 반환합니다. 같은 가속 모드와 통계 포인터를 재귀 호출에 전달해 실행 모드와 계측 의미를 유지하고, 반환 반사색에는 금속 albedo를 적용합니다. 깊이 제한은 무한 반사와 스택 증가를 막는 명시적 종료 조건입니다.

### 답변 핵심 키워드

`MaterialType dispatch`, `remaining depth`, `명시적 종료`, `reflection formula`, `normal offset`, `secondary ray`, `mode propagation`, `stats propagation`, `albedo tint`, `default diffuse`

### 백지 구현

**구현 목표**

주어진 최근접 hit 함수와 diffuse shader를 사용해 Diffuse·Metal 두 재질을 처리하는 깊이 제한 `traceRay`를 구현한다.

**인터페이스**

```cpp
enum class MaterialType {
    Diffuse,
    Metal
};

Color traceRay(const Scene& scene,
               const Ray& ray,
               int remainingDepth,
               AccelMode mode,
               RenderStats* stats = nullptr);
```

**입력과 출력**

- 입력: 장면, 광선, 남은 반사 횟수, 교차 모드, 선택적 통계.
- 출력: 해당 경로에서 계산된 색상.

**반드시 만족해야 할 조건**

- miss이면 장면 background를 반환한다.
- Diffuse는 깊이를 소비하지 않고 diffuse shader로 보낸다.
- Metal은 남은 깊이가 없으면 새 광선을 만들지 않는다.
- Metal 반사 시 깊이를 정확히 한 번 감소시킨다.
- 반사광선은 표면에서 자기 교차를 줄이는 위치에서 시작한다.
- 재귀 호출에 같은 `AccelMode`와 통계 대상을 전달한다.
- secondary ray 수는 실제로 만든 경우에만 증가한다.
- 반환 색상에 Metal albedo를 적용하는 시점이 일관되어야 한다.

**경계 조건**

- `remainingDepth == 0`인 Metal hit.
- `remainingDepth == 1`에서 한 번 반사 후 또 Metal을 맞음.
- Metal 반사광선이 background로 빠짐.
- Diffuse만 있는 기존 장면.
- normal과 입사 방향이 정면·비스듬한 경우.
- 잘못된 음수 깊이가 내부 API로 들어오는 경우의 방어 정책.

**실패 조건**

- 깊이를 감소시키지 않아 무한 재귀한다.
- 깊이가 0인데 secondary ray 통계를 증가시킨다.
- 반사 원점이 표면에 그대로 있어 자기 교차한다.
- 재귀에서 가속 모드나 통계가 사라진다.
- Diffuse 경로의 기존 출력이 재질 확장 때문에 달라진다.

**제약**

- 굴절, rough metal, Fresnel, 에너지 보존 모델은 구현하지 않는다.
- 재질 enum은 두 값만 다룬다.
- parser와 CLI 구현은 범위 밖이지만 허용 깊이 계약과 기본값은 설명할 수 있어야 한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] miss가 background를 반환하는가.
- [ ] Diffuse 장면의 결과가 깊이 값 변화에 불필요하게 흔들리지 않는가.
- [ ] 깊이 0 Metal이 secondary ray를 만들지 않는가.
- [ ] 깊이 1 Metal이 정확히 한 번의 secondary ray를 만드는가.
- [ ] 반사 방향이 입사·normal 평면 안에 있고 기대한 대칭을 이루는가.
- [ ] 반사 원점 offset이 자기 교차를 줄이는가.
- [ ] mode와 stats가 재귀 전체에 전달되는가.
- [ ] 동일 입력을 반복했을 때 결과와 카운터가 결정적인가.
- [ ] 최대 재귀 깊이와 추가 공간이 O(`remainingDepth`)로 제한되는가.

### 구현 후 설명할 것

1. `remainingDepth` 의미와 off-by-one을 피한 방법.
2. 깊이 소진 시 반환 정책과 시각적 trade-off.
3. enum dispatch를 택한 단순성 및 확장성 한계.
4. 반사 방향·원점 보정·albedo 적용 순서.
5. 기능 확장 뒤 기존 diffuse 결과를 회귀 검사해야 하는 이유.

### 원본 확인 위치

- Thread 06
- `feat(material): metal 모델과 깊이 제한 반사 구현`
- `feat(parser): 선택적 도형 재질 문법 추가`
- `test(material): 재질 파싱과 반사 깊이 검증`
- Thread 12의 `feat(cli): 반사 깊이 option과 기본값 추가`
- `include/ray/material.hpp`
- `src/scene.cpp`
- `src/shading.cpp`
- `src/parser.cpp`
- `tests/material_tests.cpp`
- `src/main.cpp`
- `src/renderer.cpp`
- `MaterialType`, `Material`, `traceRay`, `RenderSettings::maxDepth`
- 관련 Thread: 05의 직접광, 09의 secondary-ray 통계 및 결정성, 12의 `--max-depth 0..32`, 13의 재질 회귀 gate
