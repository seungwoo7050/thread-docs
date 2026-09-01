# 이미지 무결성·출력 복구·CLI 계약 워크북

이 문서는 P15~P17을 다룬다. 메모리 크기 계산과 데이터 invariant를 먼저 검증하고, 실패해도 기존 파일을 보존하는 persistence 경계, 마지막으로 이 실패를 사용자에게 안정된 종료 코드로 노출하는 CLI 경계를 연결한다.

---

<a id="p15"></a>
## [Thread 11 / `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지`] 이미지 크기 산술과 저장소 invariant

### 면접 질문

`pixels(static_cast<size_t>(width * height * 3), 0)`가 왜 위험한지 설명해 보세요. 최종 결과를 `size_t`로 cast했는데도 signed overflow를 막지 못하는 이유와, 이미지 차원·저장소 크기·pixel offset을 어떤 순서로 검증해야 하는지 말해 보세요.

꼬리 질문:

- `width`나 `height`가 음수일 때 곧바로 `size_t`로 cast하면 어떤 일이 생깁니까?
- `width * height`와 그 결과에 `3`을 곱하는 두 단계를 각각 검사해야 하는 이유는 무엇입니까?
- 나눗셈 기반 사전 검사에서 0으로 나누지 않으려면 어떤 선행 조건이 필요합니까?
- `(y * width + x) * 3`에서 cast 위치가 왜 중요합니까?
- `Image::validate()`가 pixel 저장소가 짧은 경우뿐 아니라 긴 경우도 거부해야 하는 이유는 무엇입니까?
- output 파일을 열기 전에 `validate()`해야 기존 파일 보존과 어떤 관계가 있습니까?
- `width`·`height`·`pixels`가 public인 값 객체 설계와 invariant를 생성자에서만 보장하는 설계가 충돌하는 지점은 무엇입니까?

### 30초 모범 답변

곱셈은 cast보다 먼저 `int`로 평가되므로, `width * height * 3`이 overflow한 뒤 `size_t`로 바꿔도 이미 undefined behavior입니다. 먼저 차원이 양수인지 확인하고 `size_t`로 변환한 뒤, `max / height`와 `max / 3` 형태로 각 곱셈이 가능한지 사전 검사해야 합니다. Pixel offset도 각 피연산자를 넓은 타입으로 바꾼 뒤 계산하고 좌표 범위를 보장해야 합니다. `Image::validate()`는 예상 크기와 실제 vector 크기가 정확히 같은지 확인해 직렬화와 checksum의 out-of-bounds 또는 데이터 누락을 막습니다. 이 검증을 파일을 열거나 truncate하기 전에 해야 잘못된 메모리 상태가 기존 출력 파일까지 손상시키지 않습니다.

### 답변 핵심 키워드

`overflow before cast`, `signed UB`, `positive dimensions`, `size_t conversion`, `pre-multiplication division check`, `exact storage invariant`, `safe offset`, `validate before side effect`, `short and excess storage`

### 백지 구현

**구현 목표**

양수 이미지 차원에서 RGB 저장소 크기와 pixel offset을 overflow 없이 계산하고, 이미지의 메타데이터와 저장소가 정확히 일치하는지 검증한다.

**인터페이스**

```cpp
std::size_t pixelStorageSize(int width, int height);

struct Image {
    int width;
    int height;
    std::vector<unsigned char> pixels;

    Image(int widthValue, int heightValue);
    void validate() const;

    std::size_t pixelOffset(int x, int y) const;
};
```

**입력과 출력**

- 입력: 이미지 너비·높이, pixel 좌표.
- 출력: `width * height * 3`에 해당하는 안전한 저장소 크기와 RGB 첫 byte offset.
- 실패: 유효하지 않은 차원·overflow·범위 밖 좌표·저장소 불일치에 대해 명시적 예외.

**반드시 만족해야 할 조건**

- 너비와 높이는 양수여야 한다.
- 모든 곱셈은 수행 전에 `size_t` 범위를 넘지 않음이 증명되어야 한다.
- 생성된 vector 크기는 정확히 예상 크기여야 한다.
- `validate()`는 짧거나 긴 저장소를 모두 거부한다.
- offset은 유효 좌표에서 항상 `pixels.size() - 3` 이하이고 RGB 세 byte를 가리킨다.
- signed 정수 곱셈 overflow를 일으키지 않는다.
- 실패가 발생하면 부분적으로 잘못된 vector를 외부에 성공 객체로 노출하지 않는다.

**경계 조건**

- 너비·높이가 1인 최소 이미지.
- 0 또는 음수 차원.
- 매우 큰 두 차원의 첫 곱셈 overflow.
- pixel 수는 맞지만 RGB 배수 곱셈에서 overflow.
- 마지막 유효 pixel 좌표.
- 음수 또는 너비·높이와 같은 범위 밖 좌표.
- 저장소에서 byte 하나를 제거하거나 추가한 상태.

**실패 조건**

- `int` 곱셈 뒤 결과만 cast한다.
- 음수 차원을 큰 `size_t`로 바꿔 거대한 할당을 시도한다.
- `validate()`가 최소 크기 이상만 검사해 excess 데이터를 허용한다.
- 좌표 범위 확인 전에 offset을 계산한다.
- invalid image를 checksum·출력 계층이 그대로 순회한다.

**제약**

- RGB 3채널, 채널당 1 byte로 고정한다.
- allocator 실패 자체를 별도 복구하지 않는다.
- `size_t`가 unsigned라는 사실을 이용하되 wraparound에 의존하지 않는다.
- 모든 helper는 O(1), 생성자는 저장소 크기에 비례하는 초기화 비용을 가진다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] `1×1`, `2×3`의 크기와 마지막 offset이 올바른가.
- [ ] 0·음수 차원이 할당 전에 거부되는가.
- [ ] 첫 곱셈과 채널 곱셈 overflow를 각각 탐지하는가.
- [ ] 계산 중 signed overflow가 전혀 없는가.
- [ ] `INT_MAX`에 가까운 입력을 실제 할당 없이 크기 helper에서 안전하게 거부할 수 있는가.
- [ ] 마지막 유효 좌표는 성공하고 바로 다음 좌표는 실패하는가.
- [ ] 짧은 저장소와 긴 저장소를 모두 거부하는가.
- [ ] invalid 상태에서 checksum이나 serializer가 호출되기 전에 실패하게 연결할 수 있는가.
- [ ] 정상 이미지에서 `validate()`가 데이터 내용을 변경하지 않는가.
- [ ] sanitizer 없이도 산술 검사가 정의된 동작만 사용하는가.

### 구현 후 설명할 것

1. cast가 overflow 방지책이 되지 않는 평가 순서.
2. 곱셈 전 나눗셈 검사와 그 전제 조건.
3. 메타데이터와 vector 크기를 exact invariant로 둔 이유.
4. offset helper에서 타입 변환과 bounds 검사의 순서.
5. 유효성 검증을 I/O side effect보다 먼저 배치한 이유.

### 원본 확인 위치

- Thread 11
- `fix(image): 이미지 할당과 픽셀 인덱스 overflow 방지`
- `test(image): 잘못된 차원과 저장 크기 계산 검증`
- `fix(output): 불일치한 이미지 저장소 거부`
- `test(output): 잘못된 이미지 저장소 처리 검증`
- `include/ray/renderer.hpp`
- `src/renderer.cpp`
- `src/output.cpp`
- `tests/core_tests.cpp`
- `Image`, `pixelStorageSize`, `Image::validate`
- 관련 Thread: 09의 병렬 pixel offset, 11의 PPM·checksum 입력 계약, 13의 sanitizer gate

---

<a id="p16"></a>
## [Thread 11 / `fix(output): PPM 출력 실패 시 기존 파일 보존`] 실패 안전한 파일 교체와 RAII 임시 파일

### 면접 질문

기존 `output.ppm`을 바로 `std::ofstream(path, trunc)`로 열어 직렬화하면 중간 쓰기 실패 시 어떤 데이터 손실이 생깁니까? 이 프로젝트에서 stream 직렬화와 path persistence를 분리하고, 같은 목적지 경로를 기반으로 한 임시 파일·flush/close·replace·RAII cleanup 순서를 둔 이유를 설명해 보세요.

꼬리 질문:

- 이미지 유효성 검증을 임시 파일 생성보다 먼저 해야 하는 이유는 무엇입니까?
- `operator<<` 호출이 끝났다는 사실만으로 쓰기 성공을 보장할 수 없는 이유는 무엇입니까?
- `flush()`와 `close()` 실패를 확인해야 하는 이유는 무엇입니까?
- 임시 파일을 목적지와 같은 디렉터리에 두는 것이 rename 원자성과 어떤 관계가 있습니까?
- POSIX `rename`과 Windows 교체 API의 계약 차이를 어떻게 감쌉니까?
- replace 성공 뒤에만 `commit()`해야 하는 이유는 무엇입니까?
- 프로세스 crash까지 고려한 durability와 함수 예외 안전성은 어떻게 다릅니까?
- 임시 파일 이름 생성에서 충돌·권한·symlink 문제를 더 강하게 다루려면 무엇이 필요합니까?

### 30초 모범 답변

대상 파일을 먼저 truncate하면 직렬화나 디스크 쓰기가 중간에 실패했을 때 새 파일도 완성되지 않고 기존 정상 파일도 사라집니다. 그래서 입력 이미지를 먼저 검증하고, serializer는 `ostream`에 쓰도록 분리한 뒤 목적지와 같은 디렉터리의 임시 파일에 전체 내용을 기록해 flush와 close 성공까지 확인합니다. 그다음 플랫폼별 replace helper로 목적지를 교체하고 성공한 뒤에만 임시 파일 RAII 객체를 commit합니다. 어느 단계에서 예외가 나도 미commit 임시 파일은 destructor가 정리하고 기존 목적지는 건드리지 않습니다. 이 계약은 함수 수준의 strong exception safety에 가깝지만, 디렉터리 fsync까지 하지 않으면 전원 손실 durability까지 보장한다고 말하면 안 됩니다.

### 답변 핵심 키워드

`truncate hazard`, `validate first`, `serialize to ostream`, `same-directory temporary`, `stream state`, `flush and close`, `atomic replacement`, `RAII cleanup`, `commit after replace`, `destination preservation`, `exception safety vs durability`

### 백지 구현

**구현 목표**

주어진 writer가 만든 전체 내용을 임시 파일에 성공적으로 기록한 경우에만 목적지 파일을 교체하는 `writeAtomically`를 구현한다. 플랫폼별 교체 함수는 제공되거나 별도 helper로 추상화할 수 있다.

**인터페이스**

```cpp
using StreamWriter = std::function<void(std::ostream&)>;

void writeAtomically(const std::filesystem::path& destination,
                     const StreamWriter& writer);
```

**입력과 출력**

- 입력: 목적지 경로와 stream에 전체 내용을 기록하는 callback.
- 출력: 성공 시 목적지에 완성된 새 파일.
- 실패: 예외를 전달하고 기존 목적지 내용·타입을 가능한 한 보존하며 생성한 임시 파일을 정리한다.

**반드시 만족해야 할 조건**

- writer 호출 전 목적지 파일을 truncate하지 않는다.
- 임시 파일은 목적지와 같은 디렉터리에 만든다.
- 파일 열기, writer, stream 상태, flush, close 실패를 모두 실패로 처리한다.
- replace가 성공하기 전에는 목적지를 새 파일로 간주하지 않는다.
- replace 성공 후에만 임시 파일 cleanup 책임을 해제한다.
- 예외 경로에서 자신이 만든 임시 파일을 남기지 않는다.
- 기존 목적지가 없던 경우와 있던 경우를 모두 처리한다.
- 오류 메시지는 어느 단계와 경로에서 실패했는지 식별할 수 있어야 한다.

**경계 조건**

- 목적지 파일이 없음.
- 정상 파일이 이미 있음.
- 목적지가 디렉터리여서 교체가 실패함.
- 임시 파일을 열 수 없음.
- writer가 중간에 예외를 던짐.
- stream buffer가 쓰기를 거부함.
- flush 또는 close에서 실패함.
- 임시 이름 충돌.
- replace 성공 직후 cleanup 객체 destructor가 실행됨.

**실패 조건**

- 목적지를 먼저 열어 기존 내용을 잃는다.
- writer 예외 뒤 임시 파일이 남는다.
- stream의 fail/bad 상태를 확인하지 않고 replace한다.
- replace 실패 뒤 기존 파일을 삭제하거나 변경한다.
- replace 전에 cleanup을 commit해 실패 시 임시 파일이 남는다.
- 다른 filesystem의 임시 디렉터리를 사용해 rename 계약을 잃는다.
- 함수 수준 보존을 crash durability로 과장한다.

**제약**

- 파일 잠금과 여러 프로세스의 동시 writer 조정은 구현하지 않는다.
- 디렉터리 fsync까지 요구하지 않는다.
- 안전한 임시 파일 생성 API가 제공되지 않는 환경에서는 충돌 감소 전략과 한계를 설명한다.
- destructor는 예외를 던지지 않아야 한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 정상 writer가 기존 파일을 완성된 새 내용으로 교체하는가.
- [ ] 목적지가 없을 때 새 파일을 만드는가.
- [ ] writer가 즉시·중간에 실패해도 기존 내용이 그대로인가.
- [ ] stream failure가 예외로 전달되는가.
- [ ] replace 실패 시 목적지 타입과 내용이 보존되는가.
- [ ] 모든 실패 경로에서 생성한 임시 파일이 정리되는가.
- [ ] 성공 뒤 임시 파일이 남지 않는가.
- [ ] 임시 파일이 목적지와 같은 directory에 있는가.
- [ ] cleanup destructor가 noexcept 동작을 유지하는가.
- [ ] 반복·동시 호출 시 임시 이름 충돌 가능성과 대응을 설명할 수 있는가.
- [ ] crash consistency를 주장하지 않고 보장 범위를 정확히 설명하는가.

### 구현 후 설명할 것

1. serialization과 persistence를 분리한 테스트 가능성.
2. 기존 파일 보존을 위한 단계 순서와 commit point.
3. RAII 임시 파일이 정상·예외 경로를 통합하는 방식.
4. 플랫폼별 replace를 추상화한 이유.
5. atomic visibility, exception safety, durability의 차이와 현재 보장 범위.

### 원본 확인 위치

- Thread 11
- `fix(output): PPM 출력 실패 시 기존 파일 보존`
- `test(output): 출력 실패의 대상 보존과 정리 검증`
- `include/ray/output.hpp`
- `src/output.cpp`
- `tests/output_tests.cpp`
- `writePpm(const Image&, std::ostream&)`, `writePpm(const Image&, const std::string&)`, `TemporaryOutput`, `replaceFile`
- 관련 Thread: 11의 `Image::validate`와 checksum, 12의 runtime error exit code, 13의 Windows/Linux 검증

---

<a id="p17"></a>
## [Thread 12 / `test(cli): 렌더링 옵션과 오류 종료 계약 검증`] 엄격한 옵션 parser와 종료 코드 경계

### 면접 질문

이 CLI가 알 수 없는 옵션, 중복 옵션, 값 누락, 음수·overflow 숫자를 모두 usage error로 분류하고 종료 코드 2를 반환하며, 장면 로드·렌더·출력 중 예외는 종료 코드 1로 분리한 이유를 설명해 보세요. `--threads N|auto`, `--max-depth 0..32`, `--accel linear|bvh`의 parsing 계약을 구체적으로 말해 보세요.

꼬리 질문:

- `std::stoull("4x")`가 값을 반환할 수 있는데 왜 parse 위치를 확인해야 합니까?
- 숫자 앞에 `-`가 붙은 값을 unsigned 변환 함수에 맡기기 전에 문자 검사를 하는 이유는 무엇입니까?
- `std::isdigit`에 plain `char`를 직접 넘기면 왜 위험할 수 있습니까?
- 같은 옵션이 두 번 나오면 마지막 값을 우선하는 것보다 거부하는 정책의 장점은 무엇입니까?
- `--threads auto`를 내부에서 0으로 표현하는 것과 실제 worker 0개의 의미를 어떻게 구분합니까?
- `--max-depth 0`을 허용하면서 `--threads 0`을 거부하는 이유는 무엇입니까?
- usage는 stdout과 stderr 중 어디에 쓰며, `--help`와 잘못된 사용의 종료 코드를 같게 둘 필요가 있습니까?
- shell contract test에서 stdout, stderr, 파일 생성, checksum을 함께 확인해야 하는 이유는 무엇입니까?

### 30초 모범 답변

CLI는 외부 API이므로 문법 오류와 실행 중 실패를 안정된 종료 코드로 구분했습니다. 위치 인자가 부족하거나 옵션이 알 수 없고, 중복되거나 값이 없고, 숫자가 전체 token을 소비하지 않거나 범위를 벗어나면 parser가 실패해 usage를 stderr에 출력하고 2를 반환합니다. `--threads`는 `auto` 또는 1 이상의 `unsigned int` 범위만 허용하고 내부 0은 자동 선택 sentinel입니다. `--max-depth`는 종료 조건 자체를 시험할 수 있도록 0부터 32까지 허용합니다. 문법을 통과한 뒤 load·render·write에서 난 예외는 메시지와 함께 1, 성공은 0이며 checksum 옵션은 stdout 계약을 유지합니다.

### 답변 핵심 키워드

`CLI as API`, `usage error 2`, `runtime error 1`, `success 0`, `full-token numeric parse`, `unsigned-char isdigit`, `duplicate rejection`, `sentinel auto`, `bounded depth`, `stderr vs stdout`, `black-box contract test`

### 백지 구현

**구현 목표**

두 위치 인자와 네 옵션을 엄격하게 해석하고, parser 실패와 runtime 실패를 서로 다른 종료 코드로 노출하는 축소 CLI를 구현한다.

**인터페이스**

```cpp
struct CliOptions {
    std::string scenePath;
    std::string outputPath;
    bool printChecksum = false;
    AccelMode accelMode = AccelMode::Bvh;
    unsigned int threadCount = 0;  // 0 means auto
    int maxDepth = 4;
};

bool parseUnsigned(const std::string& token,
                   unsigned long long maximum,
                   unsigned long long& value);

bool parseCli(int argc, char** argv, CliOptions& options);

int main(int argc, char** argv);
```

**지원 문법**

```text
ray-scene-tracer <scene.rt> <output.ppm>
  [--checksum]
  [--accel linear|bvh]
  [--threads N|auto]
  [--max-depth 0..32]
```

**입력과 출력**

- 입력: `argv` token.
- 출력: 성공 시 완성된 `CliOptions`와 process exit code.
- stdout: 요청된 checksum처럼 정상 데이터만 출력.
- stderr: usage 또는 runtime 오류 진단.

**반드시 만족해야 할 조건**

- 두 위치 인자가 반드시 존재해야 한다.
- 옵션 순서는 위치 인자 뒤에서 자유롭지만 각 옵션은 최대 한 번만 허용한다.
- 알 수 없는 옵션과 값 누락은 parser 실패다.
- `--accel`은 `linear`, `bvh`만 허용한다.
- `--threads`는 `auto` 또는 1 이상 `unsigned int` 최댓값 이하의 십진수만 허용한다.
- `--max-depth`는 십진수 0~32만 허용한다.
- 숫자 token은 빈 문자열이 아니고 모든 문자가 소비되어야 한다.
- usage 오류는 2, 실행 중 예외는 1, 성공은 0으로 구분한다.
- parser 실패 시 렌더링·파일 출력 같은 side effect를 시작하지 않는다.

**경계 조건**

- 인자 없음, scene만 있음, 정확한 두 위치 인자.
- 각 옵션 단독·혼합·순서 변경.
- 같은 옵션 중복.
- 옵션 값 누락.
- `--threads 0`, `-1`, `many`, 범위를 넘는 큰 정수.
- `--threads auto`.
- `--max-depth 0`, `32`, `33`, `4x`, `-1`.
- checksum을 요청하지 않은 성공 경로의 stdout.
- runtime에서 장면 파일 또는 출력 교체가 실패함.

**실패 조건**

- 접두부 숫자만 읽어 `4x`를 4로 허용한다.
- 음수 token이 unsigned wraparound로 통과한다.
- 중복 옵션의 마지막 값이 조용히 앞 값을 덮어쓴다.
- parser 오류와 renderer 오류가 같은 종료 코드로 합쳐진다.
- usage 메시지가 stdout 정상 데이터와 섞인다.
- 잘못된 옵션인데 출력 파일을 먼저 만든다.
- `auto` sentinel 0을 실제 0 worker 실행으로 해석한다.

**제약**

- GNU short option, `--option=value`, 옵션 묶음은 지원하지 않는다.
- 외부 getopt 라이브러리는 사용하지 않는다.
- `--help`의 별도 정책은 선택 사항이지만 usage 오류와 구분한다면 명확히 설명한다.
- parser는 입력 크기에 선형 시간, 옵션 수에 상수 상태만 사용한다.

```cpp
// 직접 구현
```

### 구현 후 자가 검증

- [ ] 정상 최소 호출이 성공하고 출력 파일을 만든다고 가정한 경로로 넘어가는가.
- [ ] 인자 부족·unknown option·중복·값 누락이 모두 2인가.
- [ ] usage가 stderr에 일정한 prefix로 출력되는가.
- [ ] `--threads`의 0·음수·문자·overflow가 거부되는가.
- [ ] `auto`가 내부 sentinel로 변환되고 renderer가 이를 자동 선택으로 해석하는가.
- [ ] `--max-depth`의 0과 32는 성공하고 경계 밖은 실패하는가.
- [ ] 숫자 뒤 trailing 문자가 거부되는가.
- [ ] 옵션 순서를 바꿔도 같은 `CliOptions`가 나오는가.
- [ ] runtime 예외는 1이고 usage를 다시 출력하지 않는가.
- [ ] 성공은 0이고 checksum 요청 시 stdout 형식이 다른 진단과 섞이지 않는가.
- [ ] parser 실패 경로에서 출력 경로가 생성·변경되지 않는가.

### 구현 후 설명할 것

1. 문법 오류와 runtime 오류를 다른 종료 코드로 둔 API 판단.
2. 숫자 parser의 문자 사전 검사·전체 소비·상한 검사를 나눈 이유.
3. 중복 옵션을 거부해 configuration ambiguity를 없앤 이유.
4. `threadCount == 0` sentinel과 유효 worker 수의 경계.
5. 함수 단위 parser test와 shell 수준 black-box contract test를 함께 둔 이유.

### 원본 확인 위치

- Thread 12
- `feat(cli): 장면 렌더링 명령 연결`
- `feat(cli): 가속 방식 선택 option 추가`
- `feat(cli): 작업자 수 option 추가`
- `feat(cli): 반사 깊이 option과 기본값 추가`
- `test(cli): 렌더링 옵션과 오류 종료 계약 검증`
- `src/main.cpp`
- `tests/cli_contract.sh`
- `CliOptions`, `parseUnsigned`, `parseCli`, `printUsage`, `main`
- 관련 Thread: 06의 반사 깊이, 08의 `AccelMode`, 09의 worker 수, 11의 output 실패, 13의 CLI·결정성 gate
