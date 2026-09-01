# 도메인 불변식·식별자·공개 데이터 의미

수치 정확도, 의미 식별자, immutable fact, 완전 구간과 ledger 경계를 다룬다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P01. [Thread 02 / `feat(price): define decimal comparisons`] 금액 비교의 수치 정확도와 불가능 상태 배제

**우선순위:** S

### 면접 질문

- 가격 비교에 이진 부동소수점이 아니라 `Decimal`을 사용한 이유는 무엇인가요?
- 현재가·기준가·기준일의 사용 가능 여부를 단순 `None` 조합으로 두지 않고 명시적 상태로 모델링한 이유는 무엇인가요?
- 차액의 부호, 변화율의 부호, 방향 값 사이에 어떤 invariant가 있어야 하나요?
- 꼬리 질문: 전역 Decimal context 대신 지역 context와 `ROUND_HALF_UP`을 선택할 때의 장단점은 무엇인가요?
  - **모범답변:** **프로젝트에서는** 현재가와 기준가의 자릿수 중 큰 값에 여유 정밀도 16자리를 더한 `localcontext` 안에서 비율을 계산하고, 마지막에만 0.1 단위 `ROUND_HALF_UP`을 적용합니다. 그래서 프로세스의 전역 context에 영향을 주거나 그 설정에 따라 결과가 달라지지 않고, 일반 사용자가 기대하는 절반 올림 규칙도 고정됩니다. **일반적으로는** 연산마다 context와 반올림 정책을 명시해야 하는 비용이 있지만, 금액처럼 재현성이 중요한 값에서는 그 비용보다 격리성과 결정성이 큽니다.
- 꼬리 질문: 기준 가격이 0이거나 NaN/Infinity, 허용 소수 자릿수를 넘는 값이 들어오면 어디에서 막아야 하나요?
  - **모범답변:** **프로젝트에서는** recent parser가 양수·유한·scale 0 가격만 만들고, `PriceSnapshot`과 `ReferencePrice` 값 객체가 계산 전에 같은 계약을 재검증하며, 저장 시 `DecimalField` 정밀도와 양수 DB 제약이 마지막 우회 경로를 막습니다. 따라서 0인 기준가로 나누거나 비유한 값으로 계산하는 경로가 열리지 않습니다. **일반 원칙은** 신뢰 경계에서 빠르게 실패시키되, ORM을 우회한 쓰기까지 고려해 저장 계층에도 핵심 invariant를 중복 적용하는 것입니다.

### 30초 모범 답변

가격은 재현 가능한 십진 산술이 필요하므로 `Decimal`과 명시적 반올림 규칙을 사용했습니다. 사용 가능한 기준값은 값·기준일·변화량이 모두 있어야 하고, 사용 불가 상태는 관련 수치가 없어야 하도록 상태와 필드를 함께 검증합니다. 차액과 변화율의 부호는 방향 값과 항상 일치해야 하며, 0·비유한 값·허용 정밀도 초과는 계산 전에 실패시켜 부분 결과가 남지 않게 합니다.

### 답변 핵심 키워드

`Decimal`, `ROUND_HALF_UP`, `localcontext`, `XOR 상태`, `finite`, `scale`, `sign invariant`, `fail-closed`

### 백지 구현

**구현 목표**

현재 가격과 주·월·년 기준 가격을 받아 결정적인 비교 결과를 생성하는 순수 함수를 구현한다. 사용할 수 없는 기준값은 수치 계산을 하지 않고 명시적인 결과 상태로 보존한다.

**인터페이스 또는 함수 시그니처**

```python
def compare_price_snapshot(snapshot: PriceSnapshot) -> tuple[PriceComparison, ...]:
    from datetime import date
    from decimal import ROUND_HALF_UP, Decimal, localcontext

    periods = (ComparisonPeriod.WEEK, ComparisonPeriod.MONTH, ComparisonPeriod.YEAR)
    references = tuple(snapshot.references)

    def require_price(value: Decimal, field: str) -> None:
        if not isinstance(value, Decimal) or not value.is_finite():
            raise PriceValidationError(f"{field} must be a finite Decimal")
        if (
            value <= 0
            or value.as_tuple().exponent != 0
            or len(value.as_tuple().digits) > 12
        ):
            raise PriceValidationError(
                f"{field} must be positive and fit the scale-zero price precision"
            )

    # 모든 입력을 먼저 검증해야 뒤쪽 기간의 실패로 부분 결과가 생기지 않는다.
    require_price(snapshot.current_price, "current_price")
    if len(references) != len(periods):
        raise PriceValidationError("exactly one WEEK, MONTH, and YEAR reference is required")
    by_period: dict[ComparisonPeriod, ReferencePrice] = {}
    for reference in references:
        if reference.period not in periods or reference.period in by_period:
            raise PriceValidationError("reference periods must be unique and complete")

        if reference.value_status is ValueStatus.AVAILABLE:
            if reference.value is None or reference.unavailable_reason is not None:
                raise PriceValidationError("available reference state is inconsistent")
            require_price(reference.value, "reference value")
        elif reference.value_status is ValueStatus.UNAVAILABLE:
            if reference.value is not None or not isinstance(reference.unavailable_reason, str):
                raise PriceValidationError("unavailable reference state is inconsistent")
            if not reference.unavailable_reason.strip():
                raise PriceValidationError("unavailable_reason must not be blank")
        else:
            raise PriceValidationError("unknown reference value status")

        if not isinstance(reference.reference_date_status, ReferenceDateStatus):
            raise PriceValidationError("unknown reference date status")
        date_is_provided = reference.reference_date_status is ReferenceDateStatus.PROVIDED
        if (reference.reference_date is not None) != date_is_provided:
            raise PriceValidationError("reference date state is inconsistent")
        if reference.reference_date is not None and type(reference.reference_date) is not date:
            raise PriceValidationError("reference_date must be a date")
        by_period[reference.period] = reference

    if set(by_period) != set(periods):
        raise PriceValidationError("reference periods must be unique and complete")

    results: list[PriceComparison] = []
    for period in periods:  # 입력 순서와 무관하게 공개 순서를 고정한다.
        reference = by_period[period]
        if reference.value_status is ValueStatus.UNAVAILABLE:
            results.append(
                PriceComparison(
                    period=period,
                    current_value=snapshot.current_price,
                    reference_value=None,
                    difference=None,
                    percentage=None,
                    direction=Direction.UNAVAILABLE,
                    unavailable_reason=reference.unavailable_reason,
                    reference_date_status=reference.reference_date_status,
                    reference_date=reference.reference_date,
                )
            )
            continue

        reference_value = reference.value
        if reference_value is None:  # 위 검증이 보장하는 상태를 방어적으로 좁힌다.
            raise PriceValidationError("available reference is missing its value")
        difference = snapshot.current_price - reference_value
        direction = (
            Direction.LOWER
            if difference < 0
            else Direction.HIGHER
            if difference > 0
            else Direction.EQUAL
        )
        precision = max(
            len(snapshot.current_price.as_tuple().digits),
            len(reference_value.as_tuple().digits),
        )
        with localcontext() as context:
            context.prec = precision + 16
            percentage = (
                (difference / reference_value) * Decimal(100)
            ).quantize(Decimal("0.1"), rounding=ROUND_HALF_UP)

        results.append(
            PriceComparison(
                period=period,
                current_value=snapshot.current_price,
                reference_value=reference_value,
                difference=difference,
                percentage=percentage,
                direction=direction,
                unavailable_reason=None,
                reference_date_status=reference.reference_date_status,
                reference_date=reference.reference_date,
            )
        )
    return tuple(results)
```

**입력과 출력**

- `snapshot.current_price`: 양의 유한 `Decimal` 현재 가격
- `snapshot.references`: 주·월·년 세 기간의 기준값과 기준일 상태
- 출력: 기간 순서가 고정된 세 개의 `PriceComparison`

**반드시 만족해야 할 조건**

- 현재 가격과 사용 가능한 기준 가격은 양수·유한값이며 허용 scale을 만족해야 한다.
- 세 기간이 정확히 한 번씩 있어야 하며 누락·중복·추가 기간을 허용하지 않는다.
- 사용 가능 상태는 기준 가격과 기준일을 모두 가져야 하고, 사용 불가 상태는 관련 수치가 없어야 한다.
- 차액은 현재가-기준가이며, 변화율은 기준가를 분모로 계산하고 0.1 단위로 반올림한다.
- LOWER/EQUAL/HIGHER 방향과 차액·변화율의 부호가 일치해야 한다.
- 입력 하나가 잘못되면 결과 일부를 반환하지 않는다.

**경계 조건**

- 현재가와 기준가가 같은 경우
- 주·월·년 중 일부만 사용 불가인 경우
- 경계에 가까운 큰 금액과 반올림 절반값
- reference mapping의 삽입 순서가 다른 경우

**실패 조건**

- 0 이하, NaN, Infinity 또는 허용 정밀도 초과
- 기준가 0으로 인한 나눗셈 불가
- 상태와 값/기준일 조합이 모순되는 경우
- 방향과 산술 부호가 어긋나는 경우

**제약**

- 이진 `float`로 중간 계산하지 않는다.
- 전역 Decimal context를 변경하지 않는다.
- 20분 이내에 작성할 수 있는 크기로 제한한다.

### 구현 후 자가 검증

- [ ] 정상 경로에서 주·월·년 결과 순서가 고정되어 있다.
- [ ] 같은 가격의 차액·변화율·방향이 모두 0/EQUAL이다.
- [ ] 사용 불가 기준값이 다른 기간 계산에 영향을 주지 않는다.
- [ ] 모순된 상태가 예외 없이 통과하지 않는다.
- [ ] 반올림 결과가 실행 환경의 전역 context에 의존하지 않는다.
- [ ] 입력 객체를 변경하지 않는다.

### 구현 후 설명할 것

- `Decimal`을 선택한 정확도·재현성 이유
  - **모범답변:** 이 프로젝트의 가격은 십진수로 표현되는 source fact이고 최근 가격은 scale 0, 변화율은 scale 1이라는 계약이 있습니다. `Decimal`은 정수 차액을 정확히 유지하고 나눗셈 정밀도와 반올림을 코드로 고정할 수 있어, 이진 `float`의 표현 오차나 실행 환경별 결과 차이를 피합니다.
- 상태 enum과 nullable 필드를 함께 쓰는 이유와 XOR invariant
  - **모범답변:** `None`만으로는 값이 원천에서 누락된 것인지 아직 처리되지 않은 것인지 구분하기 어렵습니다. 원본 `ReferencePrice`는 `AVAILABLE`이면 값이 있고 사유가 없어야 하며, `UNAVAILABLE`이면 값이 없고 비어 있지 않은 사유가 있어야 합니다. 기준일도 `PROVIDED`와 날짜 존재 여부를 XOR처럼 맞춰 불가능한 조합을 생성 시점에 거부합니다.
- 반올림 시점과 정밀도 손실의 trade-off
  - **모범답변:** 차액과 중간 비율은 반올림하지 않고 충분한 지역 정밀도로 계산한 뒤, 공개 계약인 백분율 0.1 단위에서 한 번만 `ROUND_HALF_UP` 합니다. 이 방식은 표시 단위보다 작은 정보는 잃지만, 중간 반올림이 누적되는 문제를 피하고 같은 입력에 같은 저장·표시 값을 만듭니다.
- 검증을 계산 전단에 두어 부분 결과를 막는 이유
  - **모범답변:** 세 기간 중 앞의 두 기간을 계산한 뒤 마지막 기간에서 실패하면 호출자가 불완전한 묶음을 사실로 오인할 수 있습니다. 따라서 스냅샷 생성자가 현재가와 세 기간의 완전성·상태 조합을 모두 확인하고, 비교 함수는 검증된 입력 전체에 대해 한 번에 tuple을 반환하도록 해 all-or-nothing 경계를 만듭니다.

### 원본 확인 위치

- Thread 02
- 커밋 `feat(price): define decimal comparisons`
- 파일 `grocery/pricing.py`
- 구성 요소 `ReferencePrice`, `PriceSnapshot`, `PriceComparison`, `compare_snapshot`
- 연관 Thread 04, 06, 08

## P02. [Thread 02 / `feat(price): identify exact retail series`] 문자열 코드 기반 의미 식별자와 교차 소스 동일성

**우선순위:** A

### 면접 질문

- 품목 코드와 시장 코드를 정수로 저장하지 않고 문자열로 보존해야 하는 이유는 무엇인가요?
- 표시 이름이 바뀌어도 동일한 계열을 식별하려면 canonical identity에 무엇을 포함하고 무엇을 제외해야 하나요?
- 최근 자료의 조사 범위와 역사 자료의 교차 소스 식별자를 분리한 이유는 무엇인가요?
- 꼬리 질문: canonical hash만 저장할 때 충돌, 직렬화 모호성, 버전 변경은 어떻게 다루겠습니까?
  - **모범답변:** **프로젝트에서는** 해시만 두지 않고 `PriceSeriesKey`의 원본 의미 필드, 그 키를 가리키는 one-to-one 관계, unique constraint, `code_manifest_sha256`과 evidence revision을 함께 저장하며 `clean()`에서 해시를 다시 계산합니다. canonical JSON은 key 정렬과 구분자 규칙으로 경계를 고정합니다. **일반적으로는** 계약 버전을 해시 입력에 포함하고 원본 필드도 보존해 충돌 시 필드 단위로 재검증하며, 직렬화 규칙 변경은 새 버전으로 병행 계산한 뒤 명시적으로 마이그레이션합니다.

### 30초 모범 답변

외부 코드에는 선행 0이 의미를 가지므로 문자열로 보존해야 합니다. 계열 식별자는 품목·품종·등급·단위처럼 의미를 결정하는 안정적인 코드와 원문 단위를 canonical하게 직렬화하고, 바뀔 수 있는 표시명이나 최근 소스에만 해당하는 조사 범위는 교차 소스 키에서 분리합니다. 해시에는 계약 버전을 포함하고 원본 필드의 DB 제약과 함께 사용해 해시 하나만 맹신하지 않습니다.

### 답변 핵심 키워드

`semantic identity`, `leading zero`, `canonical serialization`, `hash version`, `immutable key`, `cross-source mapping`

### 백지 구현

**구현 목표**

선행 0을 보존하는 계열 식별자 입력을 canonical byte sequence로 만들고 SHA-256을 계산한다. 동일 의미 입력은 field 순서와 무관하게 같은 해시를 내고, 의미가 다른 입력은 다른 canonical 표현을 가져야 한다.

**인터페이스 또는 함수 시그니처**

```python
def canonical_series_identity(parts: SeriesIdentityParts) -> str:
    import hashlib
    import json
    import re
    import unicodedata

    values = {
        "identity_contract_version": parts.identity_contract_version,
        "product_class_code": parts.product_class_code,
        "category_code": parts.category_code,
        "item_code": parts.item_code,
        "variety_code": parts.variety_code,
        "grade_code": parts.grade_code,
        "raw_unit": parts.raw_unit,
        "raw_unit_size": parts.raw_unit_size,
    }
    maximums = {
        "identity_contract_version": 64,
        "product_class_code": 2,
        "category_code": 3,
        "item_code": 32,
        "variety_code": 32,
        "grade_code": 32,
        "raw_unit": 64,
        "raw_unit_size": 64,
    }
    code_fields = {
        "product_class_code",
        "category_code",
        "item_code",
        "variety_code",
        "grade_code",
    }
    for field, value in values.items():
        if not isinstance(value, str) or not value or value != value.strip():
            raise ValueError(f"{field} must be a non-empty, trimmed string")
        if len(value) > maximums[field]:
            raise ValueError(f"{field} exceeds its maximum length")
        if any(unicodedata.category(char).startswith("C") for char in value):
            raise ValueError(f"{field} contains a control character")
        # 문자열 상태에서 ASCII 숫자를 검사해 '01'을 '1'로 정규화하지 않는다.
        if field in code_fields and re.fullmatch(r"[0-9]+", value) is None:
            raise ValueError(f"{field} must contain only ASCII digits")

    canonical = json.dumps(
        values,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")
    return hashlib.sha256(canonical).hexdigest()
```

**입력과 출력**

- 입력: 제품 구분·부류·품목·품종·등급 코드, 원문 단위, 단위 크기, identity contract version
- 출력: 소문자 64자리 SHA-256 문자열

**반드시 만족해야 할 조건**

- 모든 외부 코드는 문자열로 취급하고 선행 0을 그대로 보존한다.
- 필드 경계가 모호하지 않은 canonical 직렬화를 사용한다.
- 표시 이름과 최근 소스 전용 coverage 값은 교차 소스 identity에서 제외한다.
- 빈 필드·제어 문자·허용 길이 초과를 거부한다.
- identity contract version을 해시 입력에 포함한다.

**경계 조건**

- `01`과 `1`이 서로 다른 코드인 경우
- 한글 단위와 숫자 문자열이 함께 있는 경우
- 동일 데이터가 다른 mapping 삽입 순서로 들어오는 경우

**실패 조건**

- 정수 변환으로 선행 0이 사라지는 경우
- 구분자 충돌로 서로 다른 필드 조합이 같은 바이트열이 되는 경우
- 버전 없는 직렬화 규칙 변경

**제약**

- 표시 이름을 동일성 판단의 유일한 근거로 사용하지 않는다.
- JSON을 사용한다면 key ordering과 Unicode/공백 규칙을 명시한다.
- 15~20분 크기의 순수 함수로 구현한다.

### 구현 후 자가 검증

- [ ] `0110253`이 `110253`으로 변형되지 않는다.
- [ ] 입력 field order가 달라도 결과가 동일하다.
- [ ] 단위 또는 등급 코드 하나가 바뀌면 결과가 바뀐다.
- [ ] 표시명만 바뀌어도 교차 소스 identity는 유지된다.
- [ ] 계약 버전이 바뀌면 결과가 바뀐다.

### 구현 후 설명할 것

- 자연키와 surrogate key를 함께 두는 이유
  - **모범답변:** 품목·품종·등급·원문 단위 같은 자연키에는 source의 의미가 들어 있어 unique constraint와 drift 검증에 필요합니다. 반면 내부 관계는 UUID surrogate key를 사용하면 긴 복합키를 모든 FK에 복제하지 않아도 됩니다. 원본 `PriceSeriesKey`는 두 역할을 함께 두고 생성 후에는 수정·삭제를 막습니다.
- 해시를 identity 그 자체가 아니라 검증 가능한 요약으로 보는 이유
  - **모범답변:** SHA-256은 비교와 manifest 연결에 편리하지만 어떤 필드가 달라졌는지 설명하지 못하고 직렬화 계약에도 의존합니다. 그래서 원본은 의미 필드를 보존하고 `price_series_identity_sha256()`으로 재계산한 값과 저장값을 대조합니다. 해시는 원본 필드와 제약으로 검증 가능한 요약이지, 원본을 버릴 근거가 아닙니다.
- 최근 coverage와 교차 소스 계열 identity를 분리한 이유
  - **모범답변:** 최근 API의 `PriceSeriesKey` unique identity에는 `coverage_identity`가 포함되지만, 역사 API를 잇는 해시는 제품 구분·부류·품목·품종·등급·원문 단위만 포함합니다. 따라서 최근 수집 범위라는 source 전용 속성이 달라도 같은 소매 계열이라는 검토된 교차 소스 관계를 유지할 수 있습니다.
- 코드-이름 drift를 별도 evidence로 검증하는 방식
  - **모범답변:** parser는 관측한 품목 identity를 검토된 `ExactIdentityRegistry`에 대조하고, 지역·시장도 `(region_code, market_code)`에 대응하는 이름을 registry와 비교합니다. 모델에는 표시 이름과 evidence revision, 역사 계열에는 code manifest hash를 남기므로 이름 변경은 자연키를 조용히 바꾸지 않고 검토 증거의 변경으로 탐지됩니다.

### 원본 확인 위치

- Thread 02, Thread 08
- 커밋 `feat(price): identify exact retail series`
- 커밋 `feat(history): identify exact retail sources`
- 파일 `grocery/historical_identity_models.py`
- 구성 요소 `PriceSeriesKey`, `HistoricalRetailSeriesKey`, `RetailRegionKey`, `RetailMarketKey`, `price_series_identity_sha256`

## P03. [Thread 02 / `feat(price): store current retail snapshots`] 불변 스냅샷의 idempotent 생성과 충돌 검출

**우선순위:** A

### 면접 질문

- 같은 수집 결과가 재시도될 때 중복 row를 만들지 않으면서도 다른 내용의 재사용은 어떻게 막았나요?
- 왜 “있으면 그냥 반환”이 아니라 기존 내용과 후보 내용을 전부 대조해야 하나요?
- 애플리케이션의 `save()` 제한만으로 불변성을 보장할 수 없는 이유는 무엇인가요?
- 꼬리 질문: 동시 요청 두 개가 같은 idempotency key로 들어오면 어떤 DB 제약과 트랜잭션이 필요합니까?
  - **모범답변:** **프로젝트에서는** `(parse_run, series)`와 `(parse_run, series, source_effective_date)` unique constraint를 두고, `transaction.atomic` 안에서 parse run과 series를 `select_for_update`한 뒤 기존 snapshot도 잠가 비교·생성을 직렬화합니다. 같은 key의 후보가 먼저 저장되면 뒤 요청은 잠금 해제 후 기존 row의 모든 의미 필드를 비교해 동일 replay만 반환합니다. **일반 원칙은** 존재하지 않는 row 자체는 잠글 수 없으므로 공통 부모나 advisory lock처럼 항상 존재하는 직렬화 지점을 잠그고, unique constraint를 최종 race 방어선으로 두는 것입니다.

### 30초 모범 답변

스냅샷은 검증된 parse 결과의 사실 기록이므로 생성 후 수정하지 않습니다. 재시도는 동일한 식별자와 동일한 증거라면 기존 row를 반환하지만, 같은 key에 다른 가격·날짜·source hash가 들어오면 충돌로 실패합니다. 서비스의 원자적 조회·생성과 unique constraint, DB 불변성 guard를 함께 사용해 ORM 우회나 동시 race에서도 규칙을 유지합니다.

### 답변 핵심 키워드

`idempotency key`, `compare-before-return`, `immutable snapshot`, `unique constraint`, `transaction`, `replay conflict`

### 백지 구현

**구현 목표**

메모리 저장소 또는 제공된 repository 인터페이스 위에서 immutable snapshot을 idempotent하게 생성하고, 동일 key의 내용 충돌을 검출한다.

**인터페이스 또는 함수 시그니처**

```python
def get_or_validate_snapshot(
    candidate: SnapshotCandidate,
    store: SnapshotStore,
) -> SnapshotRecord:
    import re
    from datetime import date, datetime
    from decimal import Decimal

    if candidate.parse_run.status != "VALIDATED":
        raise ValueError("a validated parse run is required")
    if type(candidate.source_effective_date) is not date:
        raise ValueError("source_effective_date must be a date")
    if candidate.source_recorded_at is not None and not isinstance(
        candidate.source_recorded_at, datetime
    ):
        raise ValueError("source_recorded_at must be a datetime or None")
    price = candidate.current_price
    if (
        not isinstance(price, Decimal)
        or not price.is_finite()
        or price <= 0
        or price.as_tuple().exponent != 0
        or len(price.as_tuple().digits) > 12
    ):
        raise ValueError("current_price must be a positive scale-zero Decimal")
    if (
        not isinstance(candidate.source_row_sha256, str)
        or re.fullmatch(r"[0-9a-f]{64}", candidate.source_row_sha256) is None
    ):
        raise ValueError("source_row_sha256 is invalid")
    if not isinstance(candidate.source_contract_revision, str) or not candidate.source_contract_revision:
        raise ValueError("source_contract_revision must not be empty")

    semantic_values = {
        "source_effective_date": candidate.source_effective_date,
        "source_recorded_at": candidate.source_recorded_at,
        "current_price": candidate.current_price,
        "currency": "KRW",  # 원본 모델처럼 통화는 후보 입력이 아니라 고정 계약이다.
        "source_row_sha256": candidate.source_row_sha256,
        "source_contract_revision": candidate.source_contract_revision,
    }

    with store.transaction():
        # 원본처럼 항상 존재하는 parse run과 series를 먼저 잠가 최초 생성 race를 직렬화한다.
        store.lock_parse_run(candidate.parse_run.id)
        store.lock_series(candidate.series.id)
        existing = store.get_for_update(candidate.idempotency_key)
        if existing is not None:
            if any(
                getattr(existing, field) != value
                for field, value in semantic_values.items()
            ):
                raise ValueError("idempotency key was reused with different facts")
            return existing

        # create와 evidence 연결은 같은 transaction 안에서 전부 성공하거나 롤백된다.
        return store.create(candidate)
```

**입력과 출력**

- 입력: series, source date, current price, parse/artifact 증거, idempotency key를 가진 후보
- 출력: 새로 생성되었거나 동일 replay로 확인된 `SnapshotRecord`

**반드시 만족해야 할 조건**

- 후보가 허용된 parse 상태와 양의 가격을 가진 경우만 저장한다.
- key가 없으면 정확히 한 번 생성한다.
- key가 있으면 모든 의미 필드와 evidence를 대조한 뒤 동일할 때만 기존 row를 반환한다.
- 충돌 시 기존 row를 변경하지 않는다.
- 저장 도중 실패해도 부분 row가 노출되지 않는다.

**경계 조건**

- 완전히 동일한 replay
- 가격만 다른 replay
- source hash 또는 조사일만 다른 replay
- 두 실행이 동시에 같은 key를 생성하는 경우

**실패 조건**

- 미검증 parse에서 생성 시도
- 0 이하 가격 또는 잘못된 날짜
- 동일 key에 다른 증거가 연결됨
- 생성과 evidence 연결 사이의 부분 실패

**제약**

- 기존 snapshot을 update하지 않는다.
- repository가 제공하는 원자적 생성/잠금 primitive만 사용한다.
- 15~20분 내 구현 가능한 축소 문제로 푼다.

### 구현 후 자가 검증

- [ ] 첫 호출은 한 row만 만든다.
- [ ] 동일 replay는 row 수를 늘리지 않는다.
- [ ] 충돌 replay가 기존 값을 덮어쓰지 않는다.
- [ ] 실패 후 재호출 시 저장소가 일관된 상태다.
- [ ] 동시 생성 시 최종 row가 하나이며 둘 다 모순된 성공을 반환하지 않는다.

### 구현 후 설명할 것

- idempotency와 deduplication의 차이
  - **모범답변:** deduplication은 비슷해 보이는 레코드를 사후에 하나로 합치는 정책일 수 있지만, idempotency는 같은 작업 key와 같은 의미 입력을 재실행해도 최초 실행과 동일한 결과를 돌려주는 계약입니다. 이 프로젝트는 `(parse_run, series)`를 재시도 경계로 삼고, 내용이 다르면 중복으로 흡수하지 않고 replay conflict로 실패시킵니다.
- unique constraint만으로 semantic replay 검증이 끝나지 않는 이유
  - **모범답변:** unique constraint는 같은 key의 두 번째 insert를 막을 뿐, 두 번째 요청의 가격·조사일·source row hash·contract revision이 기존 사실과 같은지는 설명하지 않습니다. 그래서 원본 `get_or_validate`는 기존 row를 잠근 뒤 모든 의미 필드를 비교하고, 완전히 같을 때만 기존 객체를 반환합니다.
- 애플리케이션·ORM·DB 세 층 방어의 역할 분담
  - **모범답변:** 애플리케이션 서비스는 validated parse만 허용하고 잠금·semantic replay 비교를 담당합니다. 모델의 `full_clean`, `save`·`delete` 제한은 정상 ORM 경로의 타입·상태·불변성을 지킵니다. DB의 unique/check constraint와 UPDATE/DELETE 거부 trigger는 `bulk_create`, queryset update, 직접 SQL 같은 우회 경로와 동시 race를 막습니다.
- append-only가 정정 비용을 높이는 대신 감사 가능성을 높이는 trade-off
  - **모범답변:** snapshot을 수정할 수 없으므로 오류 정정은 올바른 artifact·parse run·새 snapshot을 다시 생성하고 publication을 교체해야 해 비용이 큽니다. 대신 과거에 어떤 source row와 parser 계약에서 어떤 가격이 나왔는지가 덮어써지지 않아 재현과 감사가 가능하고, 잘못된 값의 조용한 변조를 막습니다.

### 원본 확인 위치

- Thread 02
- 커밋 `feat(price): store current retail snapshots`
- 파일 `grocery/migrations/0004_retail_price_snapshot.py`
- 구성 요소 `RetailPriceSnapshot`, `RetailPriceSnapshot.get_or_validate`
- 연관 Thread 04, 05, 10

## P04. [Thread 08 / typed fact·precision 관련 커밋(제목은 현재 기록에서 확인되지 않음)] provider 범위 사실과 소수 정밀도의 보존

**우선순위:** A

### 면접 질문

- 월별·지역별 provider의 최저/평균/최고를 애플리케이션이 다시 계산하지 않고 그대로 저장한 이유는 무엇인가요?
- 시장별 단일 관측값을 지역 평균으로 재구성하지 않은 이유는 무엇인가요?
- DB DecimalField 정밀도와 parser 정밀도 검사를 둘 다 두어야 하는 이유는 무엇인가요?
- 꼬리 질문: `low <= mean <= high`는 모델·서비스·DB 중 어디에서 검증해야 하나요?
  - **모범답변:** **프로젝트에서는** parser의 `require_price_range`가 source row를 받을 때 먼저 실패시키고, 모델의 `full_clean`과 `provider_low <= provider_mean <= provider_high` DB check constraint가 저장 우회를 막으며, public read도 활성 publication의 손상된 사실을 다시 검출합니다. **일반적으로는** 관계 invariant를 한 곳에만 맡기기보다 신뢰 경계에서 빠른 오류를 주고 DB에서 권위 있게 보장하되, 읽기 경계 검증은 이미 손상된 데이터나 운영 drift가 공개되는 것을 차단하는 용도로 둡니다.

### 30초 모범 답변

각 API는 서로 다른 의미의 정본을 제공하므로 월별·지역별 범위와 시장 관측을 섞거나 재계산하지 않습니다. provider가 준 두 자리 소수 값을 손실 없이 보존하고, 범위형 사실은 `low <= mean <= high`, 관측형 사실은 양수라는 서로 다른 invariant를 적용합니다. parser에서 빠르게 실패시키고 DB 제약으로 우회 경로까지 막으며 source row hash와 계약 revision을 함께 남깁니다.

### 답변 핵심 키워드

`typed fact`, `provider authority`, `precision preservation`, `range invariant`, `source row hash`, `no inference`

### 백지 구현

**구현 목표**

월별·지역별 범위형 가격을 검증하는 값 객체를 구현한다. 시장 관측값과 범위형 가격을 같은 타입으로 오인하지 않도록 인터페이스를 분리한다.

**인터페이스 또는 함수 시그니처**

```python
def validate_provider_range(
    low: Decimal, mean: Decimal, high: Decimal, *, currency: str
) -> ProviderPriceRange:
    def validate_value(value: Decimal, field: str) -> None:
        if not isinstance(value, Decimal) or not value.is_finite() or value <= 0:
            raise ValueError(f"{field} must be a finite positive Decimal")

        _sign, digits, exponent = value.as_tuple()
        fractional_digits = max(-exponent, 0)
        integer_digits = max(len(digits) + exponent, 0)
        # 원본 DecimalField(max_digits=14, decimal_places=2)의 경계를 반올림 없이 적용한다.
        if fractional_digits > 2 or integer_digits > 12 or len(digits) > 14:
            raise ValueError(f"{field} exceeds the provider price precision")

    if currency != "KRW":
        raise ValueError("only KRW provider ranges are supported")
    for field, value in (("low", low), ("mean", mean), ("high", high)):
        validate_value(value, field)
    if not low <= mean <= high:
        raise ValueError("provider range must satisfy low <= mean <= high")

    # 시장 관측은 별도 타입이므로 이 생성자에서는 provider가 준 범위만 보존한다.
    return ProviderPriceRange(low=low, mean=mean, high=high, currency=currency)
```

**입력과 출력**

- 입력: provider가 준 low/mean/high `Decimal`과 통화 코드
- 출력: 검증된 불변 `ProviderPriceRange`

**반드시 만족해야 할 조건**

- 세 값 모두 양수·유한값이며 최대 두 자리 소수 정밀도를 만족한다.
- `low <= mean <= high`를 만족한다.
- 통화는 승인된 값만 허용한다.
- 입력 값을 반올림해 억지로 허용하지 않는다.
- 시장 단일 관측을 이 함수에 넣어 범위로 추정하지 않는다.

**경계 조건**

- low=mean=high
- 정수처럼 보이지만 Decimal scale이 다른 값
- 최대 허용 자릿수 경계

**실패 조건**

- 평균이 범위 밖인 경우
- 0·음수·비유한 값
- 세 자리 이상 소수 또는 허용 총 자릿수 초과
- 지원하지 않는 통화

**제약**

- provider 값을 재계산하거나 보간하지 않는다.
- 15분 이내 값 객체 또는 validator로 구현한다.
- 입력 값을 문자열 포맷팅 결과로 검증하지 않는다.

### 구현 후 자가 검증

- [ ] 두 자리 소수가 그대로 보존된다.
- [ ] 역전된 범위가 예외 없이 통과하지 않는다.
- [ ] 경계가 같은 범위는 허용된다.
- [ ] 시장 관측과 범위 타입이 호출부에서 구분된다.
- [ ] 시간·공간 복잡도가 O(1)이다.

### 구현 후 설명할 것

- source별 정본 역할을 분리한 이유
  - **모범답변:** 월별·지역별 API는 provider가 계산한 low/mean/high 범위를 주고 시장 API는 특정 시장의 단일 관측을 줍니다. 원본은 이를 `MonthlyRegionalRetailPrice`, `DailyRegionalRetailPrice`, `DailyMarketRetailPrice`로 분리해 저장하므로, 애플리케이션이 시장 관측을 재집계해 provider의 지역 평균인 것처럼 바꾸지 않습니다.
- parser 검증과 DB constraint의 중복이 방어적 중복인 이유
  - **모범답변:** parser는 세 값의 양수성, 최대 두 자리 소수, 최대 금액, 순서를 source row 문맥의 구체적인 오류로 빠르게 보고합니다. DB는 `DecimalField(14, 2)`, 양수·범위·KRW check로 parser나 ORM을 우회한 쓰기도 거부합니다. 같은 규칙의 중복이 아니라 서로 다른 신뢰 경계를 방어하는 역할 분담입니다.
- 자동 반올림 대신 fail-closed를 택한 이유
  - **모범답변:** 세 자리 소수를 두 자리로 반올림하면 provider가 실제로 준 fact와 다른 값이 되고 source row hash와의 설명 가능성도 약해집니다. 이 프로젝트는 두 자리라는 source contract를 어긴 행을 parser에서 실패시키고 그대로 보존 가능한 값만 저장해, 데이터 drift를 조용히 정상화하지 않습니다.
- 재구성 가능한 파생값과 보존해야 할 원천 사실의 경계
  - **모범답변:** provider가 준 low/mean/high와 시장별 가격, source row hash는 원천 사실이므로 그대로 보존합니다. 화면의 전체 최저·최고, 차트 좌표, range meter 같은 값은 그 사실에서 결정적으로 다시 만들 수 있어 읽기 계층에서 계산합니다. 반대로 시장 가격으로 지역 평균을 새로 만드는 것은 provider fact를 재구성하는 것이 아니라 다른 의미의 추론이므로 하지 않습니다.

### 원본 확인 위치

- Thread 08
- 파일 `grocery/historical_monthly_models.py`
- 파일 `grocery/historical_daily_models.py`
- 구성 요소 `MonthlyRegionalRetailPrice`, `DailyRegionalRetailPrice`, `DailyMarketRetailPrice`
- 연관 Thread 07, 16, 17

## P05. [Thread 16 / `feat(public): build complete monthly history context`] 완전한 월 구간 선택과 결측을 잇지 않는 차트

**우선순위:** A

### 면접 질문

- 12개월·36개월·60개월 구간을 단순 최근 N개 row가 아니라 “연속된 완전 구간”으로 선택한 이유는 무엇인가요?
- 중간 월이 비었을 때 차트 선을 이어 버리면 어떤 의미 오류가 생기나요?
- 중복 월, 정렬되지 않은 입력, 지역이 섞인 입력은 어디에서 거부해야 하나요?
- 꼬리 질문: DB에서 연속성을 검증하는 방식과 애플리케이션에서 검증하는 방식의 trade-off는 무엇인가요?
  - **모범답변:** **프로젝트에서는** 활성 monthly collection의 행을 한 번의 정렬된 query로 읽고 지역별로 묶은 뒤, 애플리케이션에서 revision의 `month_max`부터 기대 월 집합을 만들어 중복과 포함 여부를 검사합니다. 이 방식은 query 수가 고정되고 달력 로직을 단위 테스트하기 쉽지만 필요한 행을 메모리로 가져옵니다. **일반적으로는** DB의 `generate_series`, window 함수, anti-join을 쓰면 전송량을 줄일 수 있는 대신 SQL이 복잡해지고 DB 의존성이 커지므로, 데이터 상한과 실행 계획을 보고 경계를 정합니다.

### 30초 모범 답변

최근 N개 row는 중간 결측이 있어도 개수만 맞을 수 있으므로 월 단위 연속성을 별도로 검증해야 합니다. 공개 가능한 구간은 끝 월부터 역산한 모든 월이 정확히 한 번 존재할 때만 선택하고, 결측이 있으면 더 짧은 완전 구간으로 낮추거나 unavailable로 처리합니다. 차트도 결측 전후를 다른 segment로 나눠 실제로 관측하지 않은 추세를 만들지 않습니다.

### 답변 핵심 키워드

`contiguous window`, `YearMonth`, `duplicate month`, `gap segment`, `no interpolation`, `complete range`

### 백지 구현

**구현 목표**

정렬되지 않은 월별 사실에서 요청된 12/36/60개월의 연속 완전 구간을 선택한다. 표시용 chart를 만든다면 결측 전후가 같은 선분으로 연결되지 않게 한다.

**인터페이스 또는 함수 시그니처**

```python
def select_complete_month_window(
    rows: Sequence[MonthlyFact], requested_months: int
) -> HistoryWindow | None:
    import re

    if requested_months not in {12, 36, 60}:
        raise ValueError("requested_months must be 12, 36, or 60")
    if not rows:
        return None

    def month_number(value: str) -> int:
        if not isinstance(value, str) or re.fullmatch(r"[0-9]{6}", value) is None:
            raise ValueError("year_month must use YYYYMM")
        year, month = int(value[:4]), int(value[4:])
        if year < 1 or not 1 <= month <= 12:
            raise ValueError("year_month is out of range")
        return year * 12 + month - 1

    def month_value(number: int) -> str:
        year, zero_based_month = divmod(number, 12)
        return f"{year:04d}{zero_based_month + 1:02d}"

    series_ids = {row.series_id for row in rows}
    region_ids = {row.region_id for row in rows}
    if len(series_ids) != 1 or len(region_ids) != 1:
        raise ValueError("monthly facts must belong to one series and region")

    by_month: dict[str, MonthlyFact] = {}
    for row in rows:
        month_number(row.year_month)
        if row.year_month in by_month:
            raise ValueError("duplicate monthly fact")
        by_month[row.year_month] = row

    end_number = max(month_number(value) for value in by_month)
    start_number = end_number - requested_months + 1
    if start_number < 12:  # 달력에는 0001-01보다 앞선 월이 없다.
        return None
    expected = [month_value(value) for value in range(start_number, end_number + 1)]
    if any(value not in by_month for value in expected):
        return None

    selected = tuple(by_month[value] for value in expected)
    return HistoryWindow(
        rows=selected,
        start_month=expected[0],
        end_month=expected[-1],
    )
```

**입력과 출력**

- 입력: 동일 계열·지역의 월별 사실과 요청 개월 수
- 출력: 오래된 달부터 정렬된 완전 구간 또는 `None`

**반드시 만족해야 할 조건**

- 요청 개월 수는 승인된 값만 허용한다.
- 각 `YYYYMM`은 유효한 월이며 구간 안에서 정확히 한 번만 등장한다.
- 최신 월을 끝점으로 이전 월을 달력 기준으로 계산한다.
- 중간 결측이 있으면 불완전 구간을 완전한 것으로 반환하지 않는다.
- 서로 다른 계열 또는 지역이 섞이면 실패한다.

**경계 조건**

- 1월에서 전년도 12월로 넘어가는 경계
- 정확히 12개만 있는 경우
- 36개월은 불완전하지만 12개월은 완전한 데이터
- 중간 한 달이 비고 이후 데이터가 다시 있는 경우

**실패 조건**

- 중복 월
- 잘못된 `YYYYMM`
- 승인되지 않은 range
- 계열·지역 혼합

**제약**

- 결측값을 보간하거나 이전 값을 복사하지 않는다.
- O(n log n) 이내로 구현한다.
- 20분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 연도 경계에서 월 감소가 정확하다.
- [ ] 입력 순서가 달라도 결과가 같다.
- [ ] 한 달 결측이 완전 구간으로 오인되지 않는다.
- [ ] 중복 월은 조용히 덮어쓰지 않는다.
- [ ] 차트 segment 생성 시 결측 전후가 이어지지 않는다.

### 구현 후 설명할 것

- row 수와 기간 완전성이 다른 개념인 이유
  - **모범답변:** 36개 row가 있어도 같은 월이 중복되거나 중간 한 달이 빠지고 더 오래된 월이 들어오면 36개월 연속 구간이 아닙니다. 원본은 종료 월에서 정확히 36개 달력 월을 역산해 기대 집합을 만들고, 중복이 없으며 기대 월이 모두 존재할 때만 complete로 봅니다.
- 완전한 구간만 공개하는 보수적 정책의 장단점
  - **모범답변:** 관측이 없는 월을 실제 추세처럼 보여 주지 않고, 모든 사용자가 동일한 12·36·60개월 기준으로 비교할 수 있다는 장점이 있습니다. 반면 일부 유용한 관측이 있어도 구간 전체가 공개 불가가 될 수 있습니다. 원본은 최소 36개월 완전 지역만 선택지에 넣고, 해당 지역에서 60개월까지 완전할 때만 60개월 옵션을 엽니다.
- 결측 표현과 보간의 의미론적 차이
  - **모범답변:** 결측은 “그 달의 사실을 갖고 있지 않다”는 상태이고 보간은 주변 값으로 새 값을 추정하는 분석입니다. 원본 `build_history_chart`는 월 번호가 연속인 run별로만 선과 범위 band를 만들고 빠진 월에는 gap marker를 두므로, 추정 정책 없이도 관측 부재를 정직하게 표현합니다.
- DB query 수를 고정하면서 애플리케이션 검증을 수행하는 방법
  - **모범답변:** 원본은 활성 monthly collection과 series로 한 번 filter하고 region을 `select_related`한 정렬 query를 materialize합니다. 이후 `by_region`으로 그룹화하고 완전성·선택 구간·표시 자료를 모두 메모리에서 계산하므로 지역 수나 월 수가 늘어도 추가 N+1 query가 생기지 않습니다.

### 원본 확인 위치

- Thread 16
- 커밋 `feat(public): build complete monthly history context`
- 파일 `grocery/historical_history_read.py`
- 구성 요소 `history_context`, `_has_complete_months`, `_select_complete_months`
- 파일 `grocery/vnext_presentation.py`; 구성 요소 `build_history_chart`, `MonthlyChartDatum`

## P06. [Thread 17 / `feat(public): build regional and market ledgers`] 일별 ledger의 동일 날짜·지역 경계와 안정적 페이지네이션

**우선순위:** A

### 면접 질문

- 지역별 평균과 시장별 관측을 같은 화면 흐름에서 제공하면서도 서로 재계산하지 않게 한 경계는 무엇인가요?
- 시장 row의 region이 선택된 region과 반드시 일치해야 하는 이유와 검증 위치는 어디인가요?
- 페이지네이션에서 안정적 정렬 키가 없으면 어떤 오류가 생기나요?
- 꼬리 질문: 선택 가능한 날짜를 bounded하게 유지하는 이유는 무엇인가요?
  - **모범답변:** **프로젝트에서는** publication revision의 `date_max`부터 최근 30일만 조회하고, regional fact와 market fact가 함께 있는 날짜의 교집합만 역순 선택지로 제공합니다. 그래서 query·응답·날짜 UI의 크기가 예측 가능하고 서로 다른 source가 비교 가능한 날짜만 노출됩니다. **일반적으로는** 사용자 입력이 곧 무제한 과거 탐색이 되지 않도록 허용 범위를 서버가 고정하면 비용과 지연의 상한을 둘 수 있습니다.

### 30초 모범 답변

지역별 범위와 시장별 관측은 source 의미가 다르므로 각각의 typed fact를 그대로 읽습니다. 시장 화면은 선택 날짜와 지역을 먼저 고정하고 모든 market row가 그 region에 속하는지 검증하며, 정렬은 공식 코드와 추가 안정 키를 포함해 페이지 이동 중 누락·중복을 막습니다. 날짜 후보와 page size를 제한해 query와 응답 크기를 예측 가능하게 유지합니다.

### 답변 핵심 키워드

`source semantics`, `region-market invariant`, `stable ordering`, `bounded dates`, `pagination`, `no aggregation`

### 백지 구현

**구현 목표**

선택한 지역과 날짜에 속하는 시장 관측을 검증하고 안정적으로 한 페이지를 구성한다.

**인터페이스 또는 함수 시그니처**

```python
def build_market_page(
    rows: Sequence[MarketFact],
    *,
    region_id: UUID,
    selected_date: date,
    page: int,
    page_size: int,
) -> LedgerPage:
    from datetime import date
    from decimal import Decimal

    max_page_size = 30  # 원본 HISTORICAL_PAGE_SIZE와 같은 공개 상한이다.
    if type(selected_date) is not date:
        raise ValueError("selected_date must be a date")
    if (
        type(page) is not int
        or type(page_size) is not int
        or page < 1
        or page_size < 1
        or page_size > max_page_size
    ):
        raise ValueError("invalid page or page_size")

    active = load_active_historical_publication()
    if active is None:
        raise ValueError("active historical publication is missing")
    published_collection_id = active.market_collection.id

    market_keys: set[tuple[UUID, str]] = set()
    collection_ids: set[object] = set()
    series_ids: set[object] = set()
    for row in rows:
        if row.collection_id != published_collection_id:
            raise ValueError("market fact is outside the active publication")
        if row.survey_date != selected_date:
            raise ValueError("market fact date does not match the selected date")
        if row.region_id != region_id or row.market.region_id != region_id:
            raise ValueError("market fact region does not match the selected region")
        if (
            not isinstance(row.provider_price, Decimal)
            or not row.provider_price.is_finite()
            or row.provider_price <= 0
        ):
            raise ValueError("market price must be finite and positive")

        key = (row.market.region_id, row.market.market_code)
        if key in market_keys:
            raise ValueError("duplicate semantic market observation")
        market_keys.add(key)
        collection_ids.add(row.collection_id)
        series_ids.add(row.series_id)

    # 호출부가 active market collection으로 한정하고, 여기서는 scope 혼입을 한 번 더 막는다.
    if len(collection_ids) > 1 or len(series_ids) > 1:
        raise ValueError("market facts must come from one published collection")

    ordered = sorted(
        rows,
        key=lambda row: (
            row.market.market_name,
            row.market.market_code,  # 문자열 정렬로 선행 0을 보존한다.
            str(row.market_id),
        ),
    )
    total = len(ordered)
    total_pages = max(1, (total + page_size - 1) // page_size)
    if page > total_pages:
        raise ValueError("requested page is out of range")
    start = (page - 1) * page_size
    selected = tuple(ordered[start : start + page_size])
    return LedgerPage(
        rows=selected,
        page=page,
        page_size=page_size,
        total_count=total,
        total_pages=total_pages,
        has_next=page < total_pages,
    )
```

**입력과 출력**

- 입력: 공개 publication에 속한 시장 사실, 선택 region/date, 1부터 시작하는 page
- 출력: 안정적으로 정렬된 `LedgerPage`와 next 여부

**반드시 만족해야 할 조건**

- 모든 row의 조사일과 region이 요청과 일치해야 한다.
- 동일 market의 중복 관측을 거부한다.
- 공식 시장 코드와 추가 tie-breaker로 total ordering을 만든다.
- page와 page_size는 양수이며 상한을 넘지 않는다.
- 지역 평균을 시장 row에서 계산하지 않는다.

**경계 조건**

- 빈 페이지
- 마지막 페이지가 page_size보다 작은 경우
- 시장명이 같지만 코드가 다른 경우
- 선행 0이 있는 market code

**실패 조건**

- 선택 지역과 다른 market row 혼입
- 잘못된 페이지 표기 또는 범위 초과
- 중복 semantic market key
- publication 밖의 row 전달

**제약**

- 시장명 문자열로 시장 유형을 추론하지 않는다.
- 전체 row 수에 비례하는 추가 DB query를 만들지 않는 설계를 설명한다.
- 15~20분 구현 크기다.

### 구현 후 자가 검증

- [ ] 같은 입력은 항상 같은 페이지 경계를 만든다.
- [ ] 페이지 사이에 중복 row가 없다.
- [ ] 다른 지역 row가 조용히 제외되지 않고 실패한다.
- [ ] 선행 0 코드가 정수 변환으로 손실되지 않는다.
- [ ] 빈 결과와 잘못된 요청이 구분된다.

### 구현 후 설명할 것

- source별 정본을 분리한 이유
  - **모범답변:** 지역 row의 low/mean/high는 regional API가 제공한 범위이고, market row의 `provider_price`는 market API의 단일 관측입니다. 원본 `regions_context`와 `markets_context`는 각각의 collection과 typed model을 직접 읽고, 시장 목록으로 지역 평균을 다시 계산하거나 지역 범위를 시장 가격으로 분해하지 않습니다.
- 페이지네이션 안정성을 위한 total order
  - **모범답변:** 이름이나 코드 하나만으로는 동률이 생길 수 있어 페이지 경계가 비결정적일 수 있습니다. 원본 query는 날짜, 시장명, 문자열 시장 코드, 마지막으로 `market_id`를 정렬 키에 포함합니다. 선택 날짜 안에서는 뒤의 세 키가 total order를 만들어 같은 immutable publication을 읽을 때 누락·중복 없는 동일 페이지를 재현합니다.
- offset 방식의 한계와 bounded data에서의 수용 이유
  - **모범답변:** offset은 앞쪽 row가 삽입·삭제되면 다음 페이지가 밀리고 깊은 페이지일수록 DB가 건너뛰는 비용이 커집니다. 이 프로젝트는 활성 publication의 사실이 불변이고 조회 날짜를 최근 30일, 한 페이지를 30개로 제한하며 이미 고정된 결과를 메모리에서 자르므로 변동 문제와 탐색 비용이 제한적입니다. 더 큰 가변 데이터라면 동일 total-order 키 기반 cursor가 낫습니다.
- 검증 실패를 빈 결과로 숨기지 않는 이유
  - **모범답변:** 다른 지역의 market이나 비양수 가격이 섞인 것은 “관측 없음”이 아니라 publication·관계 invariant 손상입니다. 원본은 이런 경우 `PublicReadIntegrityError`를 내고, 사용자가 고른 날짜·페이지가 허용 범위 밖인 경우는 `PublicParameterError`로 구분합니다. 빈 결과로 바꾸면 운영 오류를 정상적인 데이터 부재로 오인하게 됩니다.

### 원본 확인 위치

- Thread 17
- 커밋 `feat(public): build regional and market ledgers`
- 파일 `grocery/historical_daily_read.py`
- 구성 요소 `regions_context`, `markets_context`, `HISTORICAL_PAGE_SIZE`
- 연관 Thread 08, 13, 16
