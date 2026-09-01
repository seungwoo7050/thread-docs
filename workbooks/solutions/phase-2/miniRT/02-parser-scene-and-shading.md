# 파서·장면 상태·셰이딩 워크북

이 문서는 P05~P08을 다룬다. 입력을 값으로 바꾸는 경계, 장면과 가속 구조의 lifecycle, 직접광·그림자, 재질 dispatch와 반사 깊이를 하나의 흐름으로 묶었다. 백지 구현은 면접 시간에 맞게 축소했으며 원본 구현 순서나 정답 코드는 제공하지 않는다.

---

<a id="p05"></a>
## [Thread 03 / `feat(parser): 유한 수와 범위 값 해석 구현`] 엄격한 장면 문법과 진단 가능한 입력 검증

### 면접 질문

장면 파일에서 숫자를 읽을 때 `std::stod`나 `std::stoll`이 값을 하나 반환했다는 사실만으로 입력이 유효하다고 볼 수 없는 이유를 설명해 보세요. 이 프로젝트의 parser는 어떤 단계에서 유한성, token 전체 소비, 값 범위, 중복 지시어, 필수 지시어를 검증했습니까?

꼬리 질문:

- `1.0abc`, `nan`, `inf`, 빈 token을 각각 어떻게 처리해야 합니까?
  - **모범답변:** 모두 `ParseError`로 거부합니다. `1.0abc`는 변환 위치가 token 끝이 아니고, `nan`·`inf`는 `std::isfinite`가 거짓이며, 빈 token은 `std::stod` 실패를 parser 오류로 변환합니다.
- 비율, 양수 길이, FOV, RGB처럼 의미가 다른 숫자에 하나의 범용 변환 함수만 쓰면 어떤 문제가 생깁니까?
  - **모범답변:** 어휘상 올바른 숫자와 도메인상 올바른 값은 다릅니다. 프로젝트는 유한 실수 변환 뒤 비율 `[0,1]`, 양수, FOV `(0,180)`, RGB 정수 `[0,255]`를 각 의미 경계에서 별도로 검사해 진단과 계약을 명확히 합니다.
- 카메라 방향이나 원기둥 축은 성분별 near-zero가 아니라 벡터 길이로 검사해야 하는 이유가 무엇입니까?
  - **모범답변:** 필요한 조건은 각 성분의 크기가 아니라 벡터 전체가 정규화 가능한지입니다. 원본 `requireNonzeroVector`는 안정적인 `length()`가 `kEpsilon` 이하인지 검사해 방향의 기하적 크기를 직접 판정합니다.
- 오류 메시지에 source name과 line number를 보존하면 테스트와 사용자 경험에 어떤 이점이 있습니까?
  - **모범답변:** 여러 장면 파일과 긴 입력에서도 실패 위치를 즉시 찾을 수 있고, 테스트가 단순히 예외 발생뿐 아니라 정확한 진단 문맥까지 검증할 수 있습니다. `ParseError`는 이 정보를 필드와 `what()` 메시지에 함께 보존합니다.
- 파싱 중간에 일부 `Scene` 상태가 만들어진 뒤 오류가 나면 호출자에게 무엇이 노출되어야 합니까?
  - **모범답변:** 프로젝트의 `parseScene`은 로컬 `Scene`을 구성하고 모든 필수 항목 검사와 BVH 구축까지 성공한 뒤 값으로 반환합니다. 중간 예외에서는 로컬 객체가 파괴되므로 호출자에게 부분 장면이 성공 결과로 노출되지 않습니다.

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
namespace {
std::string parseMessage(const std::string& source, std::size_t line,
                         const std::string& message) {
    std::ostringstream out;
    out << source;
    if (line > 0) out << ':' << line;
    out << ": " << message;
    return out.str();
}
}  // namespace

ParseError::ParseError(std::string source, std::size_t line,
                       std::string message)
    : std::runtime_error(parseMessage(source, line, message)),
      source_(std::move(source)), line_(line) {}

const std::string& ParseError::source() const noexcept { return source_; }
std::size_t ParseError::line() const noexcept { return line_; }

double parseDoubleToken(const std::string& token,
                        const std::string& source, std::size_t line,
                        const std::string& field) {
    try {
        std::size_t parsed = 0;
        const double value = std::stod(token, &parsed);
        if (parsed != token.size() || !std::isfinite(value)) {
            throw std::invalid_argument("not a finite full token");
        }
        return value;
    } catch (const std::exception&) {
        // 변환 라이브러리의 예외를 source/line을 가진 parser 계약으로 통일한다.
        throw ParseError(source, line,
                         "invalid " + field + " value '" + token + "'");
    }
}

double parseRatio(const std::string& token, const std::string& source,
                  std::size_t line, const std::string& field) {
    const double value = parseDoubleToken(token, source, line, field);
    if (value < 0.0 || value > 1.0) {
        throw ParseError(source, line,
                         field + " must be between 0.0 and 1.0");
    }
    return value;
}

Vec3 parseVec3(const std::string& token, const std::string& source,
               std::size_t line, const std::string& field) {
    std::vector<std::string> parts;
    std::size_t start = 0;
    while (start <= token.size()) {
        const std::size_t comma = token.find(',', start);
        parts.push_back(token.substr(start, comma - start));
        if (comma == std::string::npos) break;
        start = comma + 1;
    }
    if (parts.size() != 3 || parts[0].empty() ||
        parts[1].empty() || parts[2].empty()) {
        throw ParseError(source, line, field + " must use x,y,z format");
    }

    // 모든 성분이 검증된 뒤에만 완성된 Vec3를 반환한다.
    const double x = parseDoubleToken(parts[0], source, line, field + ".x");
    const double y = parseDoubleToken(parts[1], source, line, field + ".y");
    const double z = parseDoubleToken(parts[2], source, line, field + ".z");
    return Vec3{x, y, z};
}
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
   - **모범답변:** `stod` 성공은 숫자 접두부가 있다는 뜻일 뿐입니다. 그래서 어휘 변환, token 전체 소비, 유한성까지 공통 helper가 책임지고, 비율·양수·FOV처럼 의미별 범위는 도메인 helper가 검사해 책임과 오류 메시지를 분리합니다.
2. `ParseError`가 변환 라이브러리 예외를 감싸는 장점.
   - **모범답변:** 호출자는 `invalid_argument`와 `out_of_range` 구현 세부를 알 필요 없이 하나의 parser 오류 계약을 처리합니다. 동시에 source, line, field를 보존해 진단이 안정됩니다.
3. 성분별 작은 값 검사보다 벡터 길이 검사가 적절한 경우.
   - **모범답변:** 방향과 축처럼 정규화 가능성이 조건일 때는 전체 크기가 본질입니다. 원본은 `std::hypot` 기반 길이로 `<= kEpsilon`을 검사해 성분 조합의 실제 크기를 기준으로 거부합니다.
4. 지시어 인자 수·중복·필수 항목 검증을 어느 계층에 둘지.
   - **모범답변:** token 변환 helper보다 한 단계 위인 지시어 dispatch에서 `expectCount`와 `rejectDuplicate`를 적용합니다. 파일을 모두 읽은 뒤에는 `R`, `A`, `C` 필수 지시어 존재 여부를 장면 parser가 검사합니다.
5. parser가 완성된 값만 장면에 반영하도록 만든 방법.
   - **모범답변:** 각 복합 값은 지역 변수에서 모두 파싱·검증한 뒤 생성자나 `addShape`에 전달합니다. 전체 `Scene`도 함수의 지역 객체로 만들고 필수 검사와 `buildAcceleration`이 끝난 뒤 이동 반환합니다.

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
  - **모범답변:** index가 참조하는 `shapes_`의 항목과 기하 정보가 BVH 사용 기간 동안 유효해야 합니다. 이 프로젝트는 `Scene`이 도형을 소유하고 외부에는 const 참조만 주며, 도형 추가 시 BVH를 즉시 무효화해 stale index·bounds를 쓰지 않습니다.
- `std::unique_ptr<Shape>`가 이 장면 모델에 적합한 이유는 무엇입니까?
  - **모범답변:** 서로 다른 파생 도형을 다형적으로 저장하면서 각 객체의 소유자가 정확히 하나인 `Scene`임을 타입으로 표현합니다. 장면 파괴 시 가상 destructor를 통해 각 도형도 한 번만 해제됩니다.
- 유한 bounds가 없는 평면은 BVH 구축과 탐색에서 어떻게 다뤄야 합니까?
  - **모범답변:** `Plane::bounds()`는 `nullopt`를 반환하고, build 때 해당 shape index를 `unboundedIndices_`에 둡니다. BVH node 순회가 끝난 뒤 이 목록도 같은 최근접 hit 비교에 참여시킵니다.
- 무효화된 BVH를 그대로 순회하는 것과 선형 fallback 중 correctness 차이는 무엇입니까?
  - **모범답변:** stale BVH는 새 도형을 누락하거나 이전 bounds로 잘못 prune해 실제 hit를 잃을 수 있습니다. 선형 fallback은 느리지만 현재 `shapes_` 전체를 검사하므로 correctness를 보존합니다.
- 도형의 기하 필드를 외부에서 직접 수정할 수 있다면 `addShape` 기반 무효화만으로 충분합니까?
  - **모범답변:** 충분하지 않습니다. bounds가 바뀌어도 `addShape`가 호출되지 않아 BVH가 stale해지므로, 원본처럼 기하 필드를 private으로 두고 const accessor만 제공하거나 모든 변경 API가 무효화를 수행해야 합니다.
- rebuild 실패 시 이전 BVH를 유지할지, 비활성화할지 어떤 정책을 택하겠습니까?
  - **모범답변:** 현재 프로젝트 흐름에는 `Bvh::build` 실패 복구 정책이 별도로 구현되어 있지 않습니다. 안전한 일반 정책은 build 시작 전에 ready를 false로 두고 임시 BVH에 구축한 뒤 성공 시 교체해, 실패하면 선형 fallback을 사용하거나 검증된 이전 snapshot만 명시적으로 유지하는 것입니다.

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
void Scene::addShape(std::unique_ptr<Shape> shape) {
    if (!shape) return;  // 원본 정책: null 입력은 무시한다.
    shapes_.push_back(std::move(shape));
    bvh_.clear();
    unboundedIndices_.clear();
    accelerationReady_ = false;
}

std::size_t Scene::shapeCount() const { return shapes_.size(); }

const Shape& Scene::shapeAt(std::size_t index) const {
    return *shapes_.at(index);  // 범위 밖은 std::out_of_range로 알린다.
}

void Scene::buildAcceleration() {
    std::vector<BvhPrimitive> bounded;
    unboundedIndices_.clear();
    bounded.reserve(shapes_.size());
    unboundedIndices_.reserve(shapes_.size());

    for (std::size_t i = 0; i < shapes_.size(); ++i) {
        const std::optional<Aabb> bounds = shapes_[i]->bounds();
        if (bounds && bounds->isValid()) {
            bounded.push_back({static_cast<std::uint32_t>(i), *bounds});
        } else {
            unboundedIndices_.push_back(static_cast<std::uint32_t>(i));
        }
    }
    bvh_.build(std::move(bounded));
    // bounded/unbounded snapshot이 모두 완성된 뒤에만 ready로 전환한다.
    accelerationReady_ = true;
}

bool Scene::accelerationReady() const { return accelerationReady_; }

bool Scene::intersect(const Ray& ray, double tMin, double tMax,
                      HitRecord& hit, AccelMode mode) const {
    bool found = false;
    double closest = tMax;
    std::uint32_t bestIndex = 0;
    HitRecord candidate;

    const auto testShape = [&](std::uint32_t index) {
        if (!shapes_[index]->intersect(ray, tMin, closest, candidate)) return;
        if (!found || candidate.t < closest ||
            (candidate.t == closest && index > bestIndex)) {
            found = true;
            closest = candidate.t;
            bestIndex = index;
            hit = candidate;
        }
    };

    if (mode == AccelMode::Linear || !accelerationReady_) {
        for (std::size_t i = 0; i < shapes_.size(); ++i)
            testShape(static_cast<std::uint32_t>(i));
        return found;
    }

    struct StackEntry { std::uint32_t node; double entry; };
    std::vector<StackEntry> stack;
    const auto& nodes = bvh_.nodes();
    const auto& indices = bvh_.primitiveIndices();
    if (!nodes.empty()) {
        double entry = tMin;
        if (nodes[0].bounds.intersect(ray, tMin, closest, &entry))
            stack.push_back({0, entry});
    }

    while (!stack.empty()) {
        const StackEntry current = stack.back();
        stack.pop_back();
        if (current.entry > closest) continue;
        const BvhNode& node = nodes[current.node];
        if (node.isLeaf()) {
            for (std::uint32_t i = 0; i < node.count; ++i)
                testShape(indices[node.first + i]);
            continue;
        }

        double leftEntry = tMin, rightEntry = tMin;
        const bool leftHit = nodes[node.left].bounds.intersect(
            ray, tMin, closest, &leftEntry);
        const bool rightHit = nodes[node.right].bounds.intersect(
            ray, tMin, closest, &rightEntry);
        if (leftHit && rightHit) {
            const bool leftFirst = leftEntry < rightEntry ||
                (leftEntry == rightEntry && node.left < node.right);
            const StackEntry near = leftFirst
                ? StackEntry{node.left, leftEntry}
                : StackEntry{node.right, rightEntry};
            const StackEntry far = leftFirst
                ? StackEntry{node.right, rightEntry}
                : StackEntry{node.left, leftEntry};
            stack.push_back(far);
            stack.push_back(near);
        } else if (leftHit) {
            stack.push_back({node.left, leftEntry});
        } else if (rightHit) {
            stack.push_back({node.right, rightEntry});
        }
    }

    // 무한 도형도 동일한 closest/tie 계약에 합친다.
    for (std::uint32_t index : unboundedIndices_) testShape(index);
    return found;
}
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
   - **모범답변:** `unique_ptr`는 도형 객체 수명을 Scene에 묶고 BVH는 그 소유 저장소의 index만 보관합니다. 따라서 Scene과 도형 snapshot이 BVH보다 오래 살아야 하며, 저장소 변경은 반드시 BVH를 무효화해야 합니다.
2. mutation 시 rebuild를 즉시 하는 방식과 lazy invalidation·fallback의 trade-off.
   - **모범답변:** 즉시 rebuild는 이후 조회가 항상 빠르지만 도형을 연속 추가할 때 매번 구축 비용을 냅니다. 원본의 lazy invalidation과 선형 fallback은 변경을 싸게 하고 correctness를 유지하지만 rebuild 전 조회는 느립니다.
3. stale BVH를 절대 쓰지 않게 만드는 상태 invariant.
   - **모범답변:** `addShape`는 BVH와 unbounded 목록을 지우고 `accelerationReady_ = false`로 만듭니다. `intersect`는 ready가 false면 요청 모드가 BVH여도 선형 경로만 사용하고, build가 끝난 뒤에만 true가 됩니다.
4. unbounded 도형을 별도 경로로 관리하는 이유.
   - **모범답변:** 무한 평면을 유한 AABB로 감싸면 실제 도형 일부를 누락하거나 지나치게 큰 상자로 BVH 효율을 해칩니다. 그래서 bounded primitive만 BVH에 넣고 unbounded index는 최종 최근접 경쟁에서 직접 검사합니다.
5. 외부 직접 기하 변경을 막아야 이 lifecycle이 완전해지는 이유.
   - **모범답변:** BVH는 build 시점 bounds의 snapshot입니다. 외부가 중심·반지름·축을 바꾸면 Scene이 mutation을 관찰하지 못하므로, private 필드와 const accessor로 우회를 막아야 invalidation invariant가 완전합니다.

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
  - **모범답변:** 원본은 광원 거리에서 `kRayTMin`을 뺀 값을 상한으로 사용하고, 상한이 너무 작아지지 않게 `max(kRayTMin, distance-kRayTMin)`으로 둡니다. 따라서 광원 뒤의 교차는 구간 밖입니다.
- 표면점 그대로 광선을 시작하면 self-shadow가 생기는 이유는 무엇입니까?
  - **모범답변:** 계산된 hit point가 반올림 때문에 실제 표면의 안쪽에 놓이거나 교차 계산이 `t≈0`을 다시 유효하다고 볼 수 있습니다. 이 프로젝트는 oriented normal 방향으로 `kRayTMin`만큼 원점을 옮깁니다.
- normal 방향 offset과 광선 방향 offset은 어떤 상황에서 차이가 납니까?
  - **모범답변:** 비스듬한 광선에서는 광선 방향 offset의 표면 수직 성분이 작아 같은 면에서 충분히 떨어지지 않을 수 있습니다. normal offset은 표면 밖으로 직접 이동하지만 얇거나 매우 가까운 다른 기하를 건너뛸 수 있어 scale에 맞는 값이 필요합니다.
- `dot(normal, lightDirection) <= 0`일 때 shadow ray를 생략할 수 있는 이유는 무엇입니까?
  - **모범답변:** Lambert diffuse 항은 `max(0, dot)`이므로 해당 광원의 기여가 이미 0입니다. 가려짐 여부를 계산해도 결과가 바뀌지 않아 shadow 교차를 생략할 수 있습니다.
- 광원과 hit point가 거의 같은 위치라면 어떤 처리가 필요합니까?
  - **모범답변:** 광원까지 길이가 `kEpsilon` 이하이면 방향 정규화가 불가능하거나 불안정하므로 원본은 그 광원을 건너뜁니다. 이렇게 해야 0으로 나눠 NaN을 만드는 일을 피합니다.
- 하나의 전역 epsilon이 모든 장면 scale에서 충분하지 않을 수 있는 이유는 무엇입니까?
  - **모범답변:** 고정 offset은 아주 큰 장면에서는 수치 오차보다 작아 효과가 없고, 아주 작은 장면에서는 실제 얇은 기하를 건너뛸 만큼 클 수 있습니다. 프로젝트는 단순성과 결정성을 위해 상수를 쓰지만 일반 해법은 위치 크기나 ULP에 비례한 보정을 고려할 수 있습니다.

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
bool isOccluded(const Scene& scene, const Ray& shadowRay,
                double maxDistance, AccelMode mode) {
    HitRecord ignored;
    const double lightLimit =
        std::max(kRayTMin, maxDistance - kRayTMin);
    return scene.intersect(shadowRay, kRayTMin, lightLimit,
                           ignored, mode);
}

Color shadeDiffuseLight(const Scene& scene, const HitRecord& hit,
                        const Light& light, AccelMode mode) {
    const Vec3 toLight = light.position - hit.point;
    const double distance = toLight.length();
    if (distance <= kEpsilon) return Color{};

    const Vec3 lightDirection = toLight / distance;
    const double lambert = std::max(0.0, dot(hit.normal, lightDirection));
    if (lambert <= 0.0) return Color{};

    // oriented normal 쪽으로 옮기고, t 구간도 양 끝에서 줄인다.
    const Vec3 origin = hit.point + hit.normal * kRayTMin;
    if (isOccluded(scene, Ray{origin, lightDirection}, distance, mode))
        return Color{};

    return hit.material.albedo * light.color *
           (light.brightness * lambert);
}
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
   - **모범답변:** 가림으로 인정할 물체는 표면과 점광원 사이에 있는 것뿐입니다. `tMax`를 광원 직전으로 제한하지 않으면 광원 뒤 물체도 그림자를 만들게 됩니다.
2. 원점 offset 방향과 `t_min`을 함께 사용하는 이유.
   - **모범답변:** normal offset은 시작점을 기하 표면 밖으로 옮기고, 양의 `tMin`은 그래도 남는 시작점 근처 교차를 구간에서 제외합니다. 서로 다른 수치 오차 경로를 함께 방어합니다.
3. 광원 기여를 계산하기 전에 early-out하는 조건들.
   - **모범답변:** 광원 거리가 `kEpsilon` 이하이면 방향을 만들지 않고, Lambert 내적이 0 이하이면 기여가 없으므로 shadow ray도 만들지 않습니다. 그 뒤 실제 occlusion이면 색상 누적을 생략합니다.
4. 고정 epsilon의 단순성과 scale 민감성 trade-off.
   - **모범답변:** 하나의 `kRayTMin`은 구현과 테스트가 단순하고 결과가 결정적입니다. 반면 장면 단위가 크게 달라지면 너무 작거나 커질 수 있어 scale-aware 또는 ULP 기반 offset이 더 견고할 수 있습니다.
5. 가속 모드와 무관하게 같은 occlusion 계약을 유지하는 방법.
   - **모범답변:** `isOccluded`가 동일한 `[kRayTMin, lightLimit]`을 `Scene::intersect`에 전달하고 mode만 선택합니다. Scene의 선형·BVH 경로가 같은 hit/tie 계약을 공유하므로 가림 의미도 같습니다.

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
  - **모범답변:** 카메라 광선은 `makeCameraRay`에서 정규화되고, 도형 법선은 생성자·교차 계산과 `setFaceNormal`을 통해 단위 외향 법선에서 만들어집니다. 반사식 자체는 원본처럼 방향을 다시 정규화하지 않으므로 이 생성 경로의 invariant에 의존합니다.
- `maxDepth`를 "남은 반사 횟수"와 "현재 깊이" 중 어느 의미로 정의했습니까?
  - **모범답변:** 프로젝트에서는 남은 반사 횟수입니다. Metal hit에서 0 이하면 종료하고, 실제 secondary ray를 만들 때만 `maxDepth - 1`로 재귀합니다.
- 깊이 소진 시 검정, local shading, background 중 무엇을 반환할지 정책이 중요한 이유는 무엇입니까?
  - **모범답변:** 종료 색은 장면의 밝기와 회귀 checksum에 직접 영향을 줍니다. 원본 계약은 깊이 0의 Metal에서 검정을 반환하며, 다른 선택도 가능하지만 parser 기본값·테스트·사용자 기대와 일관되어야 합니다.
- 반사 광선도 BVH/선형 모드와 통계를 그대로 전달해야 하는 이유는 무엇입니까?
  - **모범답변:** 재귀 중 모드가 바뀌면 한 렌더가 서로 다른 알고리즘을 섞어 성능 비교와 결과 동치의 의미가 깨집니다. 같은 통계 포인터를 전달해야 secondary 이후의 primitive/AABB 검사도 전체 작업량에 포함됩니다.
- parser에서 재질 생략 시 diffuse를 기본으로 하고 알 수 없는 이름을 거부한 이유는 무엇입니까?
  - **모범답변:** 기존 장면 문법과 결과를 유지하면서 선택적으로 Metal을 추가하기 위해 생략값을 Diffuse로 뒀습니다. 오타를 조용히 Diffuse로 해석하지 않도록 알려지지 않은 이름은 `ParseError`로 거부합니다.
- enum dispatch와 가상 재질 객체 방식의 trade-off는 무엇입니까?
  - **모범답변:** 두 재질뿐인 프로젝트에서는 enum 분기가 소유권과 호출 흐름이 단순하고 데이터도 작습니다. 재질 종류가 많아지면 분기 함수가 커지므로 가상 함수나 variant가 확장에는 유리할 수 있지만 구조와 간접 호출 비용이 늘어납니다.

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
Color traceRay(const Scene& scene, const Ray& ray, int remainingDepth,
               AccelMode mode, RenderStats* stats) {
    HitRecord hit;
    if (!scene.intersect(ray, kRayTMin,
                         std::numeric_limits<double>::infinity(),
                         hit, mode, stats)) {
        return scene.background;
    }

    if (hit.material.type == MaterialType::Diffuse) {
        return shadeHit(scene, hit, ray, mode, stats);
    }

    // 프로젝트 계약에서 depth는 남은 secondary ray 횟수다.
    if (remainingDepth <= 0) return Color{};

    const Vec3 reflected = ray.direction -
        hit.normal * (2.0 * dot(ray.direction, hit.normal));
    const Ray reflectedRay{
        hit.point + hit.normal * kRayTMin,
        reflected
    };
    if (stats) ++stats->secondaryRays;

    return hit.material.albedo *
        traceRay(scene, reflectedRay, remainingDepth - 1, mode, stats);
}
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
   - **모범답변:** 값을 앞으로 만들 수 있는 secondary ray 수로 정의했습니다. Metal hit에서 먼저 `<= 0`을 확인하고, 광선을 실제 생성한 한 지점에서만 counter 증가와 `-1`을 함께 수행합니다.
2. 깊이 소진 시 반환 정책과 시각적 trade-off.
   - **모범답변:** 원본은 깊이가 소진된 Metal을 검정으로 처리해 종료 규칙이 명확합니다. 깊은 반사면이 어두워지는 단점이 있으며 background나 local shading을 택할 수도 있지만 프로젝트 결과와는 달라집니다.
3. enum dispatch를 택한 단순성 및 확장성 한계.
   - **모범답변:** 현재 Diffuse와 Metal 두 경로는 `MaterialType` 한 번의 분기로 명확히 나뉩니다. 재질이 늘면 `traceRay`가 모든 타입을 알아야 해 open/closed 측면의 확장성은 떨어집니다.
4. 반사 방향·원점 보정·albedo 적용 순서.
   - **모범답변:** oriented normal로 반사 방향을 계산하고, 같은 normal 쪽으로 원점을 옮긴 뒤 재귀 결과에 현재 Metal의 albedo를 곱합니다. 따라서 miss한 반사광도 background를 받은 뒤 금속 색으로 tint됩니다.
5. 기능 확장 뒤 기존 diffuse 결과를 회귀 검사해야 하는 이유.
   - **모범답변:** parser 기본 재질과 dispatch 기본 경로가 바뀌면 Metal이 없는 기존 장면도 달라질 수 있습니다. 기존 diffuse 장면의 pixel과 checksum을 비교해야 확장이 이전 계약을 보존했음을 확인할 수 있습니다.

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
