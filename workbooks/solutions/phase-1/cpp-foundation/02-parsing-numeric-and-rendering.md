# 파싱·수치 경계·결정적 출력 면접 워크북

이 문서는 입력 문자열을 먼저 명확한 의미 모델로 분류하고, 정의되지 않은 signed 산술을 피하며, 출력 환경과 무관한 결과를 만드는 문제를 다룬다.

<a id="a-02"></a>
## [Thread 05 / `feat(scalar): scalar 리터럴 문법과 종류 분류`, `feat(scalar): locale 고정 수치 추출과 경계 보존`] scalar 리터럴 문법 파서

### 면접 질문

하나의 문자열을 문자, 유한 실수, NaN, 양·음의 무한대로 분류해야 합니다. 라이브러리 변환 함수에 문자열 전체를 바로 맡기지 않고 문법을 먼저 검증해야 하는 이유와, `f`, `+`, `-`, `.`처럼 문자와 숫자 문법이 겹칠 수 있는 입력의 우선순위를 어떻게 정의하겠습니까?

꼬리 질문:

- 단일 문자 `f`는 문자지만 `1.0f`는 float suffix를 가진 수치로 분류하려면 어떤 순서가 필요합니까?
  - 모범답변: 프로젝트 파서는 정확한 특수 토큰을 먼저 처리하고, 그다음 한 글자이면서 숫자가 아닌 허용 ASCII를 문자로 분류한 뒤, 나머지를 유한 수치 문법으로 검사합니다. 따라서 `f`는 문자 단계에서 끝나고 `1.0f`는 수치 문법의 suffix로 처리됩니다.
- `42f`는 거부하면서 `1e2f`, `-0.0f`를 허용하는 규칙을 어떻게 표현합니까?
  - 모범답변: 문법 검사 중 소수점 존재 여부와 지수 존재 여부를 따로 기록하고, trailing `f`를 소비한 경우 둘 중 하나가 참이어야 한다는 조건을 둡니다. 즉 suffix 허용 조건을 `has_point || has_exponent`로 명시합니다.
- locale가 소수점 문자를 바꾸더라도 `1.5`만 허용하려면 무엇을 고정해야 합니까?
  - 모범답변: 문법 단계에서 소수점을 ASCII `'.'`로 직접 검사하고, 값 추출용 `istringstream`에도 `std::locale::classic()`을 imbue해야 합니다. 전역 locale이나 호출 환경은 변경하지 않습니다.
- `-0`, `-0.0`, `-0e20`에서 부호 정보를 값 변환 뒤에도 보존해야 하는 이유는 무엇입니까?
  - 모범답변: `value == 0.0` 비교만으로는 양의 0과 음의 0을 구분할 수 없고, 변환·출력 과정에서 부호 표현이 환경에 따라 달라질 수 있습니다. 그래서 선두 `-`와 가수의 모든 숫자가 0인지 문법 단계에서 기록합니다.
- 실제 값이 0이 된 비영(非零) 입력, 예를 들어 지나치게 작은 수는 왜 문법 성공과 별개로 거부할 수 있습니까?
  - 모범답변: 문법적으로 0이 아닌 입력이 변환 결과 0이 되면 지원하는 `double` 범위에서 의미 있는 값을 보존하지 못한 underflow입니다. 이 프로젝트 API는 조용한 정보 손실 대신 파서 실패로 보고하도록 경계를 정했습니다.
- 문자열 안의 NUL, 공백, 0x80 이상 바이트를 초기에 거부하면 이후 단계가 어떻게 단순해집니까?
  - 모범답변: 이후 상태 전이는 ASCII 숫자·기호만 고려하면 되고, C 문자열 경계 혼동이나 locale별 문자 분류를 피할 수 있습니다. 전체 문자열 소비 여부도 명시된 바이트 집합 안에서 결정적으로 검사할 수 있습니다.

### 30초 모범 답변

파싱은 먼저 허용 바이트와 문법을 검증하고, 그 뒤 classic locale에서 값만 추출해야 합니다. 특수 토큰을 정확히 일치시킨 뒤, 한 글자 ASCII 비공백 비숫자는 문자로 분류하고, 나머지는 부호·가수·소수점·지수·선택적 소문자 `f`를 상태 전이로 검사합니다. `f` suffix는 소수점이나 지수가 있는 수치에만 허용해 `42f`를 막고, 입력의 가수 자릿수가 모두 0인지 별도로 기록해 음의 0을 보존합니다. 변환 결과가 overflow이거나 비영 입력이 0으로 underflow하면 문법은 맞아도 지원 범위를 벗어난 것으로 거부합니다.

### 답변 핵심 키워드

lexical validation, grammar before conversion, explicit precedence, exact special tokens, classic locale, full consumption, negative zero metadata, overflow, nonzero underflow, ASCII boundary, embedded NUL

### 백지 구현

#### 구현 목표

아래 계약을 따르는 `parseScalar`를 구현한다. 파서는 입력을 의미 모델로 분류할 뿐, `char/int/float/double` 출력은 담당하지 않는다.

#### 인터페이스 또는 함수 시그니처

```cpp
enum ScalarKind
{
    scalar_character,
    scalar_finite,
    scalar_nan,
    scalar_positive_infinity,
    scalar_negative_infinity
};

struct ParsedScalar
{
    ScalarKind kind;
    double value;
    bool float_suffix;
    bool negative_zero;
};

class ScalarParseError
{
};

ParsedScalar parseScalar(const std::string &text);

namespace
{

bool isScalarDigit(char value)
{
    return value >= '0' && value <= '9';
}

bool isScalarWhitespace(char value)
{
    return value == ' ' || value == '\t' || value == '\n' ||
           value == '\v' || value == '\f' || value == '\r';
}

void rejectScalarBytes(const std::string &text)
{
    if (text.empty())
        throw ScalarParseError();
    for (std::size_t i = 0; i < text.size(); ++i)
    {
        const unsigned char value =
            static_cast<unsigned char>(text[i]);

        if (value == 0 || value > 127 || isScalarWhitespace(text[i]))
            throw ScalarParseError();
    }
}

ParsedScalar makeSpecialScalar(ScalarKind kind, bool float_suffix)
{
    ParsedScalar scalar;

    scalar.kind = kind;
    scalar.float_suffix = float_suffix;
    scalar.negative_zero = false;
    if (kind == scalar_nan)
        scalar.value = std::numeric_limits<double>::quiet_NaN();
    else if (kind == scalar_negative_infinity)
        scalar.value = -std::numeric_limits<double>::infinity();
    else
        scalar.value = std::numeric_limits<double>::infinity();
    return scalar;
}

bool allScalarMantissaDigitsAreZero(const std::string &text)
{
    std::size_t i = 0;

    if (text[i] == '+' || text[i] == '-')
        ++i;
    while (i < text.size() && text[i] != 'e' && text[i] != 'E' &&
           text[i] != 'f')
    {
        if (isScalarDigit(text[i]) && text[i] != '0')
            return false;
        ++i;
    }
    return true;
}

void validateScalarFiniteGrammar(const std::string &text,
                                 bool &float_suffix)
{
    std::size_t i = 0;
    std::size_t integer_digits = 0;
    std::size_t fraction_digits = 0;
    bool has_point = false;
    bool has_exponent = false;

    if (text[i] == '+' || text[i] == '-')
        ++i;
    while (i < text.size() && isScalarDigit(text[i]))
    {
        ++integer_digits;
        ++i;
    }
    if (i < text.size() && text[i] == '.')
    {
        has_point = true;
        ++i;
        while (i < text.size() && isScalarDigit(text[i]))
        {
            ++fraction_digits;
            ++i;
        }
    }
    if (integer_digits == 0 && fraction_digits == 0)
        throw ScalarParseError();

    if (i < text.size() && (text[i] == 'e' || text[i] == 'E'))
    {
        std::size_t exponent_digits = 0;

        has_exponent = true;
        ++i;
        if (i < text.size() && (text[i] == '+' || text[i] == '-'))
            ++i;
        while (i < text.size() && isScalarDigit(text[i]))
        {
            ++exponent_digits;
            ++i;
        }
        if (exponent_digits == 0)
            throw ScalarParseError();
    }

    float_suffix = false;
    if (i < text.size() && text[i] == 'f')
    {
        float_suffix = true;
        ++i;
    }
    // 정수 표기의 trailing f는 의도적으로 허용하지 않는다.
    if (i != text.size() || (float_suffix && !has_point && !has_exponent))
        throw ScalarParseError();
}

double extractScalarFiniteValue(const std::string &text,
                                bool float_suffix)
{
    const std::string number =
        float_suffix ? text.substr(0, text.size() - 1) : text;
    std::istringstream input(number);
    double value;

    input.imbue(std::locale::classic());
    input >> value;
    if (input.fail() || !input.eof() || value != value ||
        value > std::numeric_limits<double>::max() ||
        value < -std::numeric_limits<double>::max())
        throw ScalarParseError();
    return value;
}

}

ParsedScalar parseScalar(const std::string &text)
{
    rejectScalarBytes(text);
    if (text == "nan" || text == "nanf")
        return makeSpecialScalar(scalar_nan, text == "nanf");
    if (text == "+inf" || text == "+inff")
        return makeSpecialScalar(scalar_positive_infinity,
                                 text == "+inff");
    if (text == "-inf" || text == "-inff")
        return makeSpecialScalar(scalar_negative_infinity,
                                 text == "-inff");

    ParsedScalar scalar;
    if (text.size() == 1 && !isScalarDigit(text[0]) &&
        static_cast<unsigned char>(text[0]) >= 33)
    {
        scalar.kind = scalar_character;
        scalar.value = static_cast<unsigned char>(text[0]);
        scalar.float_suffix = false;
        scalar.negative_zero = false;
        return scalar;
    }

    bool float_suffix;
    validateScalarFiniteGrammar(text, float_suffix);
    const bool all_zero = allScalarMantissaDigitsAreZero(text);

    scalar.kind = scalar_finite;
    scalar.float_suffix = float_suffix;
    scalar.negative_zero = text[0] == '-' && all_zero;
    if (all_zero)
        scalar.value = scalar.negative_zero ? -0.0 : 0.0;
    else
    {
        scalar.value = extractScalarFiniteValue(text, float_suffix);
        if (scalar.value == 0.0) // 문법상 비영 값의 underflow를 거부한다.
            throw ScalarParseError();
    }
    return scalar;
}
```

#### 입력과 출력

허용 입력:

- 단일 ASCII 비공백 비숫자 문자(바이트 33~127): `a`, `f`, `+`, `-`, `.`
- 정수형 수치: `0`, `9`, `+42`, `-42`
- 소수점 수치: `42.`, `.5`, `-0.0`
- 지수 수치: `1e2`, `1.e2`, `-0e20`
- `f` suffix 수치: 소수점 또는 지수가 있는 경우에만 허용, 예: `1e2f`, `-0.0f`
- 특수값: `nan`, `nanf`, `+inf`, `+inff`, `-inf`, `-inff`

출력은 분류 종류, `double` 값, float suffix 여부, 음의 0 여부다.

#### 반드시 만족해야 할 조건

- 입력 전체를 소비해야 하며 trailing garbage를 허용하지 않는다.
- 빈 입력, 모든 공백, 문자열 안의 공백, NUL, 비ASCII 바이트를 거부한다.
- 특수값은 위 목록과 정확히 일치할 때만 허용한다.
- 단일 숫자 문자는 문자보다 수치가 우선한다.
- 단일 `f`는 suffix 없는 문자다.
- 유한 수치에는 가수 자릿수가 최소 하나 있어야 한다.
- 지수 표시 뒤에는 선택적 부호와 최소 한 자리 숫자가 있어야 한다.
- `f` suffix는 소문자만 허용하고, 소수점이나 지수가 없는 정수형 표기에는 붙일 수 없다.
- 수치 추출은 classic locale를 사용한다.
- `-0`, `-0.0`, `-0e20`, `-0.0f`는 `negative_zero == true`다.
- 표현 범위를 넘는 값과 비영 입력이 0으로 변환되는 underflow를 거부한다.

#### 경계 조건

- `0`, `-0`, `+0`, `.0`, `0.`, `0e999`
- `f`, `9`, `+`, `-`, `.`
- `1e+1`, `1e-1`, `1.e2`, `.5e2`
- `42f`, `42F`, `1.0F`
- `nan`, `nanf`, `+nan`, `inf`
- `1e309`, `1e-9999`
- `std::string("4\0", 2)`

#### 실패 조건

- 잘못된 바이트·문법·범위는 모두 `ScalarParseError`로 보고한다.
- 변환 스트림이 일부 문자만 읽었거나 fail 상태가 되면 실패다.
- 문법상 0이 아닌 가수인데 변환 결과가 0이면 지원하지 않는 underflow로 실패다.

#### 필요한 제약

- C++98만 사용한다.
- 정규식과 `std::strtod`에 문법 검증 전체를 위임하지 않는다.
- 문자 분류에 locale 의존 `std::isdigit(char)`를 직접 사용하지 않는다.
- 파싱 결과를 문자열 출력으로 만들지 않는다.

### 구현 후 자가 검증

- [ ] 허용된 단일 문자와 숫자 한 글자의 우선순위가 다르다.
- [ ] `f`와 `1.0f`가 서로 다른 종류로 분류된다.
- [ ] `42f`와 대문자 suffix가 거부된다.
- [ ] 소수점 앞이나 뒤 중 하나에만 숫자가 있어도 가수 전체에는 숫자가 존재한다.
- [ ] 지수 숫자가 없는 `1e`, `1e+`가 거부된다.
- [ ] 특수 토큰의 부호·suffix 조합을 정확히 제한한다.
- [ ] 공백·NUL·비ASCII 입력을 값 변환 전에 거부한다.
- [ ] 호출 환경의 locale가 달라도 점을 소수점으로 사용한다.
- [ ] 음의 0 네 가지 표기에서 부호 메타데이터가 보존된다.
- [ ] overflow와 비영 underflow를 문법 오류와 동일한 파서 실패 경계에서 처리한다.
- [ ] 입력 길이에 대해 시간 복잡도가 선형이다.

### 구현 후 설명할 것

1. 문법 검사와 값 추출을 분리해 허용 언어가 라이브러리 구현에 흔들리지 않게 한 이유
   - 모범답변: stream 변환기는 구현과 locale에 따라 선행 기호·부분 소비·특수 표기를 다르게 받을 수 있습니다. 프로젝트는 먼저 ASCII 상태 전이로 허용 언어를 고정하고 stream은 이미 검증된 수치의 값 추출만 맡겨 문법 계약을 독립시켰습니다.
2. 특수값, 단일 문자, 유한 수치의 우선순위를 선택한 근거
   - 모범답변: 특수값은 정확한 다문자 토큰이므로 먼저 확정하고, 한 글자 비숫자는 문자 계약에 따라 그다음 처리합니다. 숫자 한 글자는 문자보다 유한 수치가 우선하며, 나머지는 전체 유한 문법을 통과해야 합니다.
3. `42f` 금지 규칙을 단순 trailing `f` 허용보다 명시적으로 모델링한 방식
   - 모범답변: 파서가 `has_point`와 `has_exponent`를 기록하고 suffix를 소비한 뒤 `float_suffix && !has_point && !has_exponent`를 오류로 판정합니다. 따라서 suffix 자체뿐 아니라 그 앞 표기의 형태가 계약에 포함됩니다.
4. 음의 0을 `double` 값 비교만으로 재구성하지 않고 메타데이터로 보존한 이유
   - 모범답변: 0과 -0은 동등 비교에서 같고, 변환 과정의 부호 보존에 의존하면 원래 표기 의도를 안정적으로 복원하기 어렵습니다. 선두 부호와 가수 숫자를 검사해 `negative_zero`를 저장하면 렌더러가 결정적으로 `-0.0`을 출력할 수 있습니다.
5. underflow를 허용할지 오류로 볼지에 대한 API trade-off
   - 모범답변: 0으로 반올림된 값을 허용하면 더 넓은 입력을 받지만 원래 비영 의미를 조용히 잃습니다. 이 프로젝트는 scalar 변환 결과의 의미 보존을 택해 비영 underflow를 오류로 보며, 근사 계산 API라면 경고나 상태 플래그와 함께 0을 허용할 수도 있습니다.

### 원본 확인 위치

- Thread 05
- 커밋: `feat(scalar): scalar 리터럴 문법과 종류 분류`
- 커밋: `feat(scalar): locale 고정 수치 추출과 경계 보존`
- 커밋: `test(scalar): literal 문법과 수치 범위 검증`
- 파일: `src/ScalarLiteral.hpp`, `src/ScalarLiteral.cpp`, `tests/test_scalar_literal.cpp`
- 함수·구조체: `cppf::scalar_detail::ScalarLiteral`, `parseScalarLiteral`, `rejectInvalidBytes`, `validateFiniteGrammar`
- 관련 Thread: 06, 10, 14

---

<a id="a-03"></a>
## [Thread 06 / `feat(scalar): 부동소수점 표현과 원자 출력 구현`] 수치 투영과 결정적 원자 렌더링

### 면접 질문

이미 파싱된 scalar를 `char`, `int`, `float`, `double` 네 줄로 출력합니다. 변환 가능성 판단과 실제 cast를 어떻게 분리하고, 호출자가 설정한 stream locale·precision·flags에 결과가 영향받지 않으며, 파싱·렌더링 실패 시 출력에 앞부분이 남지 않도록 어떻게 구성하겠습니까?

꼬리 질문:

- `char`의 범위 가능성과 displayable 여부를 왜 별도로 구분합니까?
  - 모범답변: 0~127 미만 값은 ASCII 문자 코드로 투영할 수 있지만, 0~31과 127은 제어 문자라 화면에 안전하게 표시할 수 없습니다. 그래서 범위 밖은 `impossible`, 범위 안 비출력 문자는 `Non displayable`로 의미를 나눕니다.
- `127.9`를 정수 127로 투영할 수 있고 `128`은 문자로 투영할 수 없는 기준은 무엇입니까?
  - 모범답변: C++의 유한 실수→정수 변환은 0 방향 절단이므로 원래 값이 `-1`보다 크고 `128`보다 작으면 결과가 0~127입니다. `127.9`는 127이 되지만 128은 상한 밖입니다.
- int 경계에서 소수 부분을 0 방향으로 절단한다는 규칙을 범위 검사에 어떻게 반영합니까?
  - 모범답변: 원래 값이 `INT_MIN - 1`보다 크고 `INT_MAX + 1`보다 작은지 cast 전에 검사합니다. 이 열린 구간의 값만 0 방향 절단했을 때 `int` 범위에 들어갑니다.
- 유한 `double`을 `float`로 cast했을 때 0이 되면 왜 원래 값이 0인지 확인합니까?
  - 모범답변: 원래 값도 0이면 정확한 0 투영이지만 비영 값이 0이 되면 float 범위에서 underflow로 정보가 사라진 것입니다. 프로젝트는 이를 지원 불가능한 투영으로 구분해 `impossible`을 출력합니다.
- `-0.0`, NaN, 양·음의 무한대의 표기를 어떻게 고정합니까?
  - 모범답변: `negative_zero`면 stream의 부호 처리에 맡기지 않고 `-0`에서 시작해 `.0`과 suffix를 붙입니다. 특수값은 kind별로 `nanf/nan`, `+inff/+inf`, `-inff/-inf` 문자열을 직접 선택합니다.
- 임시 문자열을 완성한 뒤 한 번 `write`해도 최종 외부 I/O 자체의 부분 기록 가능성까지 완전히 없앨 수 있습니까?
  - 모범답변: 없습니다. 내부 변환·할당 실패 전에는 외부 stream을 건드리지 않는 논리적 원자성은 얻지만, 마지막 장치 write 도중의 오류나 부분 기록은 stream과 I/O 계층의 책임이며 rollback할 수 없습니다.

### 30초 모범 답변

먼저 각 대상형에 대해 변환 가능성을 판정하고, 가능할 때만 cast해 정의되지 않거나 구현 의존적인 경계를 피합니다. 문자는 ASCII 범위와 printable 범위를 나눠 `impossible`과 `Non displayable`을 구분하고, float는 범위 초과뿐 아니라 비영 값이 0으로 내려가는 underflow도 막습니다. 출력은 classic locale와 고정 precision을 가진 내부 `ostringstream`에서 네 줄 전체를 완성한 뒤 호출자 stream에 기록하면 caller의 flags·locale을 건드리지 않고, 파싱·렌더링 실패 전에는 외부 출력이 없습니다. 다만 마지막 stream write 자체의 장치 오류나 부분 기록 가능성은 별도 I/O 계층의 책임입니다.

### 답변 핵심 키워드

projection predicate, cast after proof, truncation toward zero, ASCII/displayable split, float overflow, nonzero underflow, negative zero, canonical special values, classic locale, isolated formatting state, render-then-write, I/O atomicity boundary

### 백지 구현

#### 구현 목표

`ParsedScalar`를 받아 아래 정확한 네 줄 형식으로 출력하는 함수를 구현한다. 파싱은 호출 전에 끝났다고 가정하며, 함수가 내부 렌더링을 완료하기 전에는 대상 stream에 아무것도 기록하지 않는다.

#### 인터페이스 또는 함수 시그니처

```cpp
void writeScalarProjections(const ParsedScalar &scalar,
                            std::ostream &output);

namespace
{

bool scalarIsProjectableValue(const ParsedScalar &scalar)
{
    return scalar.kind == scalar_character || scalar.kind == scalar_finite;
}

bool canProjectScalarChar(const ParsedScalar &scalar)
{
    return scalarIsProjectableValue(scalar) && scalar.value > -1.0 &&
           scalar.value < 128.0;
}

bool canProjectScalarInt(const ParsedScalar &scalar)
{
    const double lower =
        static_cast<double>(std::numeric_limits<int>::min()) - 1.0;
    const double upper =
        static_cast<double>(std::numeric_limits<int>::max()) + 1.0;

    // 열린 구간 안이면 0 방향 절단 결과가 int 범위에 든다.
    return scalarIsProjectableValue(scalar) && scalar.value > lower &&
           scalar.value < upper;
}

bool canProjectScalarFloat(const ParsedScalar &scalar)
{
    const double maximum = std::numeric_limits<float>::max();

    if (!scalarIsProjectableValue(scalar) || scalar.value < -maximum ||
        scalar.value > maximum)
        return false;
    const float value = static_cast<float>(scalar.value);
    return scalar.value == 0.0 || value != 0.0f;
}

std::string quotedScalarCharacter(int value)
{
    if (value == '\'')
        return "'\\\''";
    if (value == '\\')
        return "'\\\\'";
    return std::string("'") + static_cast<char>(value) + "'";
}

std::string finiteScalarNumber(double value,
                               bool as_float,
                               bool negative_zero)
{
    std::ostringstream rendered;
    std::string result;

    rendered.imbue(std::locale::classic());
    rendered.precision(as_float ? std::numeric_limits<float>::digits10
                                : std::numeric_limits<double>::digits10);
    if (value == 0.0 && negative_zero)
        result = "-0";
    else if (as_float)
        rendered << static_cast<float>(value);
    else
        rendered << value;
    if (result.empty())
        result = rendered.str();
    if (result.find('.') == std::string::npos &&
        result.find('e') == std::string::npos &&
        result.find('E') == std::string::npos)
        result += ".0";
    return result;
}

void renderScalarCharacter(const ParsedScalar &scalar, std::ostream &output)
{
    output << "char: ";
    if (!canProjectScalarChar(scalar))
        output << "impossible";
    else
    {
        const int value = static_cast<int>(scalar.value);
        if (value < 32 || value > 126)
            output << "Non displayable";
        else
            output << quotedScalarCharacter(value);
    }
    output << '\n';
}

void renderScalarInteger(const ParsedScalar &scalar, std::ostream &output)
{
    output << "int: ";
    if (!canProjectScalarInt(scalar))
        output << "impossible";
    else
        output << static_cast<int>(scalar.value);
    output << '\n';
}

void renderScalarFloat(const ParsedScalar &scalar, std::ostream &output)
{
    output << "float: ";
    if (scalar.kind == scalar_nan)
        output << "nanf";
    else if (scalar.kind == scalar_positive_infinity)
        output << "+inff";
    else if (scalar.kind == scalar_negative_infinity)
        output << "-inff";
    else if (!canProjectScalarFloat(scalar))
        output << "impossible";
    else
        output << finiteScalarNumber(scalar.value, true,
                                     scalar.negative_zero) << 'f';
    output << '\n';
}

void renderScalarDouble(const ParsedScalar &scalar, std::ostream &output)
{
    output << "double: ";
    if (scalar.kind == scalar_nan)
        output << "nan";
    else if (scalar.kind == scalar_positive_infinity)
        output << "+inf";
    else if (scalar.kind == scalar_negative_infinity)
        output << "-inf";
    else
        output << finiteScalarNumber(scalar.value, false,
                                     scalar.negative_zero);
    output << '\n';
}

}

void writeScalarProjections(const ParsedScalar &scalar,
                            std::ostream &output)
{
    std::ostringstream rendered;

    rendered.imbue(std::locale::classic());
    renderScalarCharacter(scalar, rendered);
    renderScalarInteger(scalar, rendered);
    renderScalarFloat(scalar, rendered);
    renderScalarDouble(scalar, rendered);
    const std::string result = rendered.str();

    // 내부 렌더링이 모두 성공한 뒤에만 caller stream을 건드린다.
    output.write(result.data(), static_cast<std::streamsize>(result.size()));
}
```

#### 입력과 출력

출력 형식:

```text
char: ...
int: ...
float: ...f
double: ...
```

표현 규칙:

- 투영할 수 없으면 `impossible`
- ASCII 범위에는 있지만 printable ASCII 32~126이 아니면 `Non displayable`
- printable quote와 backslash는 이스케이프해 따옴표로 감싼다.
- 정수처럼 보이는 유한 부동소수점 결과에는 `.0`을 붙인다.
- NaN과 무한대는 `nanf`/`nan`, `+inff`/`+inf`, `-inff`/`-inf`로 정규화한다.
- 음의 0은 `-0.0f`, `-0.0`으로 보존한다.

#### 반드시 만족해야 할 조건

- 문자 투영은 유한 값 또는 문자 값에만 허용하며 0 이상 128 미만이어야 한다.
- 문자 값으로 cast한 결과가 32~126이면 출력 가능 문자다.
- int 투영은 0 방향 절단 결과가 `int` 범위 안에 들어가는 값에만 허용한다.
- float 투영은 `float` 최대 유한 범위를 넘지 않아야 한다.
- 원래 값이 0이 아닌데 float cast 결과가 0이면 `impossible`이다.
- NaN과 무한대는 char/int에는 투영하지 않는다.
- 내부 숫자 렌더링은 classic locale와 명시적 precision을 사용한다.
- caller stream의 flags, fill, width, precision, locale를 변경하지 않는다.
- 내부 렌더링 중 예외가 발생하면 caller stream 내용은 그대로다.

#### 경계 조건

- 문자: `-1`, `-0.5`, `0`, `31`, `32`, `39`, `92`, `126`, `127`, `127.9`, `128`
- int: `INT_MIN` 부근의 소수, `INT_MAX` 부근의 소수, 범위 바로 밖
- float: `FLT_MAX` 부근, `double`로는 유효하지만 float로 overflow하는 값
- float underflow: 작은 비영 값
- `-0`, NaN, 양·음의 무한대
- caller stream이 scientific/showpos/다른 locale/작은 precision을 가진 상태

#### 실패 조건

- 지원하지 않는 투영은 함수 전체 실패가 아니라 해당 줄의 `impossible`이다.
- 내부 문자열 할당이나 렌더링 예외는 외부 stream 기록 전에 전파한다.
- 최종 `output.write` 실패는 stream 상태로 관찰될 수 있으며, 장치 수준 rollback은 요구하지 않는다.

#### 필요한 제약

- C++98만 사용한다.
- caller stream 설정을 저장했다가 복구하는 방식 대신 별도 내부 stream을 사용한다.
- `std::printf`나 전역 locale 변경을 사용하지 않는다.
- 입력 문자열을 다시 파싱하지 않는다.

### 구현 후 자가 검증

- [ ] `31`, `32`, `126`, `127`, `128`의 문자 상태가 모두 구분된다.
- [ ] quote와 backslash가 모호하지 않은 문자열로 출력된다.
- [ ] 음의 소수의 int 변환이 0 방향 절단 규칙을 따른다.
- [ ] int 최대·최소 경계 바로 안과 밖을 구분한다.
- [ ] float overflow와 float underflow를 별도로 검증한다.
- [ ] `-0.0f`와 `-0.0`의 부호가 보존된다.
- [ ] NaN·무한대 표기가 입력 suffix와 무관하게 정규화된다.
- [ ] 정수형 부동소수점 문자열에 `.0`이 붙는다.
- [ ] caller stream의 기존 flags·fill·width·precision·locale가 호출 뒤 같다.
- [ ] 내부 렌더링 실패를 주입했을 때 외부 stream에 접두부가 남지 않는다.
- [ ] 최종 write 실패와 변환 실패의 책임 경계를 설명할 수 있다.

### 구현 후 설명할 것

1. "cast한 뒤 결과를 확인"보다 "cast 전에 가능성을 증명"해야 하는 이유
   - 모범답변: 범위 밖 부동소수 값을 정수형으로 cast하는 동작은 안전한 사후 검사 수단이 아니며, signed 범위를 벗어난 변환 결과에 의존할 수 없습니다. 원래 값을 대상형의 허용 구간과 먼저 비교한 뒤 cast해야 변환 자체가 유효합니다.
2. int 투영의 범위를 원래 실수값과 절단 규칙으로 정의한 방식
   - 모범답변: 0 방향 절단을 반영해 `INT_MIN - 1 < value < INT_MAX + 1`인 유한 값만 허용합니다. 예를 들어 `INT_MAX + 0.9`처럼 표현 가능한 경우 절단 후 최대값이 되지만 정확히 `INT_MAX + 1`은 거부됩니다.
3. float underflow를 `0`과 구분해 `impossible`로 처리한 API 판단
   - 모범답변: 원래 값 0은 float 0으로 정확히 표현되지만, 비영 double이 float cast 후 0이면 의미가 소실됩니다. 프로젝트 출력은 근삿값을 조용히 내는 대신 해당 대상형 투영이 불가능하다고 명시합니다.
4. caller stream 상태를 직접 수정·복구하지 않고 내부 stream으로 격리한 이유
   - 모범답변: 저장·복구 방식은 중간 예외 때 복구 누락 가능성이 있고 width처럼 일회성 상태의 의미도 복잡합니다. 별도 classic-locale stream은 caller의 locale·flags·precision을 처음부터 읽거나 바꾸지 않습니다.
5. 논리적 원자 렌더링과 물리적 I/O atomicity의 차이
   - 모범답변: 임시 문자열 완성 전에는 외부 출력이 전혀 없어 변환 단계는 원자적으로 보입니다. 하지만 마지막 `write`가 파일·소켓·사용자 정의 streambuf에 전달된 뒤 발생한 부분 기록은 되돌릴 수 없으므로 물리적 원자성은 보장하지 않습니다.

### 원본 확인 위치

- Thread 06
- 커밋: `feat(scalar): 문자와 정수 투영 결과 출력`
- 커밋: `feat(scalar): 부동소수점 표현과 원자 출력 구현`
- 커밋: `test(scalar): 변환 가능성·출력·CLI 오류 검증`
- 파일: `include/cppf/ScalarConverter.hpp`, `src/ScalarConverter.cpp`, `tests/test_scalar_converter.cpp`
- 클래스·함수: `cppf::ScalarConverter::write`, `finiteNumber`, `canProjectChar`, `canProjectInt`, `canProjectFloat`, `writeCharacter`, `writeInteger`, `writeFloating`, `writeDouble`
- 관련 Thread: 05, 11, 14

---

<a id="s-04"></a>
## [Thread 10 / `feat(rpn): signed token과 stack 문법 처리`, `feat(rpn): overflow 검사 산술 연산 구현`] 오버플로 검사 RPN 평가기

### 면접 질문

공백으로 구분된 signed decimal 정수와 `+ - * /`만 허용하는 RPN 평가기를 구현하십시오. 모든 `long` 입력 경계를 지원하되 signed overflow나 `-LONG_MIN` 같은 정의되지 않은 동작을 일으키지 않고, 잘못된 문법과 산술 overflow를 구분해 보고해야 합니다.

꼬리 질문:

- 숫자 토큰을 `long`으로 누적하지 않고 unsigned magnitude로 읽는 이유는 무엇입니까?
  - 모범답변: 양수 한계와 음수 한계의 절댓값이 다르고, `LONG_MIN`은 양의 `long`으로 표현할 수 없습니다. 부호와 magnitude를 분리하면 unsigned 영역에서 자리 추가 전 범위를 검사하고 두 한계를 모두 처리할 수 있습니다.
- `LONG_MIN`의 절댓값을 어떻게 UB 없이 표현합니까?
  - 모범답변: 파싱 한계는 `static_cast<unsigned long>(LONG_MAX) + 1`로 만들고, 기존 음수 값의 magnitude는 `static_cast<unsigned long>(-(value + 1)) + 1`로 계산합니다. 어느 경우에도 `-LONG_MIN`을 직접 평가하지 않습니다.
- 덧셈·뺄셈은 실제 연산 전에 어떤 부등식으로 overflow를 판정합니까?
  - 모범답변: 덧셈은 `right > 0 && left > LONG_MAX - right`, `right < 0 && left < LONG_MIN - right`를 검사합니다. 뺄셈은 `right > 0 && left < LONG_MIN + right`, `right < 0 && left > LONG_MAX + right`를 검사합니다.
- 곱셈을 단순히 `result / right == left`로 검사하면 이미 overflow가 발생한 뒤인 이유는 무엇입니까?
  - 모범답변: `result`를 얻기 위한 signed 곱셈 자체가 범위를 넘는 순간 정의되지 않은 동작입니다. 사후 나눗셈은 이미 발생한 UB를 복구하지 못하므로 magnitude끼리 `left_magnitude > limit / right_magnitude`를 먼저 검사해야 합니다.
- `LONG_MIN / -1`이 특별한 이유는 무엇입니까?
  - 모범답변: 수학적 결과는 `LONG_MAX + 1`이라 `long`에 표현되지 않으며, C++ signed 나눗셈에서도 overflow입니다. 다른 0이 아닌 제수 조합은 몫이 범위 안이므로 이 조합만 별도로 차단합니다.
- 연산자를 만났을 때 오른쪽 피연산자를 먼저 pop해야 하는 이유는 무엇입니까?
  - 모범답변: RPN에서 `left right op` 순서로 push되므로 stack top은 right입니다. right를 먼저 꺼내고 그다음 left를 꺼내야 뺄셈과 나눗셈의 비가환 순서가 보존됩니다.
- 빈 식, 피연산자 부족, 남은 피연산자, 탭 구분자를 각각 어떻게 분류합니까?
  - 모범답변: 모두 `InvalidRpnExpression`입니다. 빈 식과 공백뿐인 식은 최종 stack 크기가 0이고, 피연산자 부족은 연산 시 크기 2 미만, 남은 피연산자는 종료 시 크기 1 초과이며, 탭은 space가 아닌 토큰 바이트라 숫자·연산자 어느 쪽에도 맞지 않습니다.

### 30초 모범 답변

토큰을 왼쪽부터 읽으며 숫자는 stack에 push하고, 연산자는 오른쪽과 왼쪽 피연산자를 순서대로 pop해 검사된 연산 결과를 다시 push합니다. 숫자 파싱은 부호와 unsigned magnitude를 분리하고 자리 추가 전에 `(limit - digit) / 10`으로 범위를 검사해 `LONG_MIN`까지 처리합니다. 덧셈·뺄셈은 경계 부등식을 먼저 확인하고, 곱셈은 UB 없이 얻은 두 magnitude와 결과 부호별 허용 한계를 나눠 비교합니다. 0으로 나누기는 문법/연산 오류, `LONG_MIN / -1`은 overflow이며, 입력을 다 읽은 뒤 stack 원소가 정확히 하나여야 성공입니다.

### 답변 핵심 키워드

RPN stack, tokenization, unsigned magnitude, `LONG_MIN`, precondition overflow check, operand order, division by zero, `LONG_MIN / -1`, exact final stack size, O(n) time, O(k) space

### 백지 구현

#### 구현 목표

아래 문법만 처리하는 RPN 평가기를 10~30분 안에 구현한다. 정답 코드와 같은 helper 분할은 요구하지 않으며, 정의되지 않은 signed overflow를 일으키지 않는 것이 핵심이다.

#### 인터페이스 또는 함수 시그니처

```cpp
class InvalidRpnExpression
{
};

class RpnOverflow
{
};

long evaluateRpn(const std::string &expression);

// 필요하면 다음 helper를 선언해도 된다.
bool parseLongToken(const std::string &token, long &value);
long checkedAdd(long left, long right);
long checkedSubtract(long left, long right);
long checkedMultiply(long left, long right);
long checkedDivide(long left, long right);

namespace
{

bool isRpnDigit(char value)
{
    return value >= '0' && value <= '9';
}

unsigned long rpnMagnitudeOf(long value)
{
    if (value >= 0)
        return static_cast<unsigned long>(value);
    // value + 1은 LONG_MIN에서도 표현 가능하다.
    return static_cast<unsigned long>(-(value + 1)) + 1;
}

bool isRpnOperator(const std::string &token)
{
    return token.size() == 1 &&
           (token[0] == '+' || token[0] == '-' || token[0] == '*' ||
            token[0] == '/');
}

long applyRpnOperator(long left, long right, char operation)
{
    if (operation == '+')
        return checkedAdd(left, right);
    if (operation == '-')
        return checkedSubtract(left, right);
    if (operation == '*')
        return checkedMultiply(left, right);
    return checkedDivide(left, right);
}

}

bool parseLongToken(const std::string &token, long &value)
{
    std::size_t i = 0;
    bool negative = false;
    unsigned long magnitude = 0;

    if (token.empty())
        return false;
    if (token[i] == '+' || token[i] == '-')
    {
        negative = token[i] == '-';
        ++i;
    }
    if (i == token.size())
        return false;

    unsigned long limit =
        static_cast<unsigned long>(std::numeric_limits<long>::max());
    if (negative)
        ++limit; // 음수 쪽은 LONG_MIN의 magnitude 하나를 더 허용한다.

    while (i < token.size())
    {
        if (!isRpnDigit(token[i]))
            return false;
        const unsigned long digit =
            static_cast<unsigned long>(token[i] - '0');
        if (magnitude > (limit - digit) / 10)
            throw RpnOverflow();
        magnitude = magnitude * 10 + digit;
        ++i;
    }

    if (!negative)
        value = static_cast<long>(magnitude);
    else if (magnitude == limit)
        value = std::numeric_limits<long>::min();
    else
        value = -static_cast<long>(magnitude);
    return true;
}

long checkedAdd(long left, long right)
{
    if ((right > 0 &&
         left > std::numeric_limits<long>::max() - right) ||
        (right < 0 &&
         left < std::numeric_limits<long>::min() - right))
        throw RpnOverflow();
    return left + right;
}

long checkedSubtract(long left, long right)
{
    if ((right > 0 &&
         left < std::numeric_limits<long>::min() + right) ||
        (right < 0 &&
         left > std::numeric_limits<long>::max() + right))
        throw RpnOverflow();
    return left - right;
}

long checkedMultiply(long left, long right)
{
    const bool negative = (left < 0) != (right < 0);
    const unsigned long left_magnitude = rpnMagnitudeOf(left);
    const unsigned long right_magnitude = rpnMagnitudeOf(right);
    unsigned long limit =
        static_cast<unsigned long>(std::numeric_limits<long>::max());

    if (left_magnitude == 0 || right_magnitude == 0)
        return 0;
    if (negative)
        ++limit;
    if (left_magnitude > limit / right_magnitude)
        throw RpnOverflow();

    const unsigned long product = left_magnitude * right_magnitude;
    if (!negative)
        return static_cast<long>(product);
    if (product == limit)
        return std::numeric_limits<long>::min();
    return -static_cast<long>(product);
}

long checkedDivide(long left, long right)
{
    if (right == 0)
        throw InvalidRpnExpression();
    if (left == std::numeric_limits<long>::min() && right == -1)
        throw RpnOverflow();
    return left / right;
}

long evaluateRpn(const std::string &expression)
{
    std::vector<long> stack;
    std::size_t i = 0;

    while (i < expression.size())
    {
        while (i < expression.size() && expression[i] == ' ')
            ++i;
        if (i == expression.size())
            break;

        const std::size_t start = i;
        while (i < expression.size() && expression[i] != ' ')
            ++i;
        const std::string token = expression.substr(start, i - start);

        if (isRpnOperator(token))
        {
            if (stack.size() < 2)
                throw InvalidRpnExpression();
            const long right = stack.back();
            stack.pop_back();
            const long left = stack.back();
            stack.pop_back();
            stack.push_back(applyRpnOperator(left, right, token[0]));
        }
        else
        {
            long value;
            if (!parseLongToken(token, value))
                throw InvalidRpnExpression();
            stack.push_back(value);
        }
    }
    if (stack.size() != 1)
        throw InvalidRpnExpression();
    return stack.back();
}
```

#### 입력과 출력

- 숫자 토큰은 선택적 `+` 또는 `-` 뒤에 십진 숫자가 한 자리 이상 온다.
- 연산자는 한 글자 `+`, `-`, `*`, `/`다.
- 토큰 구분자는 ASCII space 한 종류이며 여러 개를 허용한다.
- 성공하면 최종 `long` 값을 반환한다.

#### 반드시 만족해야 할 조건

- `LONG_MIN`부터 `LONG_MAX`까지 모든 유효한 정수 토큰을 읽는다.
- 범위를 벗어난 숫자 토큰은 overflow로 보고한다.
- 각 연산은 실제 signed 연산 전에 overflow 가능성을 검사한다.
- `LONG_MIN`의 magnitude를 계산할 때 직접 `-LONG_MIN`을 수행하지 않는다.
- 연산자마다 stack에 최소 두 값이 있어야 한다.
- 뺄셈과 나눗셈의 좌우 순서를 보존한다.
- 0으로 나누기는 잘못된 식으로 보고한다.
- `LONG_MIN / -1`은 overflow로 보고한다.
- 모든 토큰 처리 후 stack에 정확히 한 값만 남아야 한다.
- 탭, 줄바꿈, NUL, 비ASCII 바이트, 소수점, 지수, suffix는 허용하지 않는다.

#### 경계 조건

- `LONG_MAX`, `LONG_MIN`, 범위보다 한 자리 큰 값
- `+0`, `-0`, 선행 0
- `LONG_MAX 1 +`
- `LONG_MIN 1 -`
- `LONG_MIN -1 *`, `LONG_MIN 1 *`, `0 LONG_MIN *`
- `LONG_MIN -1 /`, `LONG_MIN 1 /`
- 음수 나눗셈의 0 방향 절단
- 빈 문자열, 공백만 있는 문자열
- `1 +`, `1 2`, `1 2 %`

#### 실패 조건

- 숫자 범위 초과와 산술 범위 초과는 `RpnOverflow`다.
- 문법 오류, 피연산자 부족·과잉, 알 수 없는 토큰, 0 나누기는 `InvalidRpnExpression`이다.
- 어떤 실패에서도 외부 상태를 수정하지 않는다.

#### 필요한 제약

- C++98만 사용한다.
- `long long`, compiler builtin overflow 함수, arbitrary precision 정수를 사용하지 않는다.
- 전체 식을 재귀 AST로 만들지 않고 stack 기반 한 번 순회로 처리한다.
- 토큰화를 위해 `operator>>`에 모든 공백 규칙을 맡기지 않는다.

### 구현 후 자가 검증

- [ ] 단일 숫자 식이 그대로 반환된다.
- [ ] 네 연산의 정상 경로와 음수 조합을 검증했다.
- [ ] 뺄셈·나눗셈이 pop 순서를 뒤집지 않는다.
- [ ] `LONG_MIN`과 `LONG_MAX` 토큰이 정확히 허용된다.
- [ ] 숫자 누적 중 overflow를 자리 추가 전에 검출한다.
- [ ] 덧셈·뺄셈의 양·음 방향 overflow를 모두 검증한다.
- [ ] 곱셈의 네 부호 조합과 0, `LONG_MIN` 경계를 검증한다.
- [ ] 0 나누기와 `LONG_MIN / -1`을 서로 다른 오류로 구분한다.
- [ ] 피연산자 부족과 남은 피연산자를 모두 거부한다.
- [ ] 탭·줄바꿈·NUL·비ASCII·소수점·지수·suffix를 거부한다.
- [ ] 시간 복잡도는 입력 길이에 선형이고 공간은 최대 stack 깊이에 비례한다.

### 구현 후 설명할 것

1. signed 범위를 부호와 unsigned magnitude로 분리한 이유
   - 모범답변: `LONG_MIN`의 절댓값은 양의 `long`으로 표현되지 않지만 `unsigned long`에는 담을 수 있습니다. 부호별 한계를 정한 뒤 magnitude를 누적하면 음수 최소값까지 UB 없이 읽고 마지막에만 signed 값으로 변환할 수 있습니다.
2. 각 산술 연산을 "연산 후 검사"가 아니라 "허용 구간 확인 후 연산"으로 만든 방식
   - 모범답변: 덧셈·뺄셈은 반대 항을 경계에서 빼거나 더한 부등식으로, 곱셈은 unsigned magnitude를 `limit / other`와 비교해 먼저 안전성을 증명합니다. 나눗셈은 0과 `LONG_MIN / -1`을 차단한 뒤에만 실제 연산합니다.
3. 곱셈에서 결과 부호에 따라 허용 magnitude가 `LONG_MAX` 또는 `LONG_MAX + 1`이 되는 이유
   - 모범답변: 양수 결과의 최대 magnitude는 `LONG_MAX`지만 음수 결과에는 `LONG_MIN`이 있어 그 magnitude가 하나 더 큽니다. 두 피연산자의 부호가 다를 때만 limit를 `LONG_MAX + 1`로 늘립니다.
4. 0 나누기와 overflow를 다른 예외 범주로 나눈 API 의미
   - 모범답변: 0 나누기는 연산 의미가 성립하지 않는 잘못된 식이므로 `InvalidRpnExpression`, `LONG_MIN / -1`은 연산은 정의되지만 대상 `long` 범위를 넘으므로 `RpnOverflow`입니다. 호출자가 입력 수정과 범위 조정을 구분할 수 있습니다.
5. 입력 전체를 한 번 순회하는 stack 평가의 시간·공간 복잡도
   - 모범답변: 각 바이트는 공백 건너뛰기나 토큰 읽기에서 상수 번 처리되므로 시간은 O(n)입니다. stack은 아직 소비되지 않은 피연산자를 보관하므로 공간은 최대 stack 깊이 O(k), 최악에는 O(n)입니다.

### 원본 확인 위치

- Thread 10
- 커밋: `feat(rpn): signed token과 stack 문법 처리`
- 커밋: `feat(rpn): overflow 검사 산술 연산 구현`
- 커밋: `test(rpn): 산술 경계와 잘못된 token 검증`
- 파일: `include/cppf/RpnEvaluator.hpp`, `src/RpnEvaluator.cpp`, `tests/test_rpn_evaluator.cpp`
- 클래스·함수: `cppf::RpnEvaluator::evaluate`, `parseLong`, `magnitudeOf`, `checkedAdd`, `checkedSubtract`, `checkedMultiply`, `checkedDivide`, `applyOperator`
- 관련 Thread: 05, 08, 11, 14
