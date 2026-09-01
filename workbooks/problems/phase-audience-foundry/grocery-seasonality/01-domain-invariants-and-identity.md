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
- 꼬리 질문: 기준 가격이 0이거나 NaN/Infinity, 허용 소수 자릿수를 넘는 값이 들어오면 어디에서 막아야 하나요?

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
    # 직접 구현
    raise NotImplementedError
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
- 상태 enum과 nullable 필드를 함께 쓰는 이유와 XOR invariant
- 반올림 시점과 정밀도 손실의 trade-off
- 검증을 계산 전단에 두어 부분 결과를 막는 이유

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
    # 직접 구현
    raise NotImplementedError
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
- 해시를 identity 그 자체가 아니라 검증 가능한 요약으로 보는 이유
- 최근 coverage와 교차 소스 계열 identity를 분리한 이유
- 코드-이름 drift를 별도 evidence로 검증하는 방식

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
    # 직접 구현
    raise NotImplementedError
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
- unique constraint만으로 semantic replay 검증이 끝나지 않는 이유
- 애플리케이션·ORM·DB 세 층 방어의 역할 분담
- append-only가 정정 비용을 높이는 대신 감사 가능성을 높이는 trade-off

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
    # 직접 구현
    raise NotImplementedError
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
- parser 검증과 DB constraint의 중복이 방어적 중복인 이유
- 자동 반올림 대신 fail-closed를 택한 이유
- 재구성 가능한 파생값과 보존해야 할 원천 사실의 경계

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
    # 직접 구현
    raise NotImplementedError
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
- 완전한 구간만 공개하는 보수적 정책의 장단점
- 결측 표현과 보간의 의미론적 차이
- DB query 수를 고정하면서 애플리케이션 검증을 수행하는 방법

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
    # 직접 구현
    raise NotImplementedError
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
- 페이지네이션 안정성을 위한 total order
- offset 방식의 한계와 bounded data에서의 수용 이유
- 검증 실패를 빈 결과로 숨기지 않는 이유

### 원본 확인 위치

- Thread 17
- 커밋 `feat(public): build regional and market ledgers`
- 파일 `grocery/historical_daily_read.py`
- 구성 요소 `regions_context`, `markets_context`, `HISTORICAL_PAGE_SIZE`
- 연관 Thread 08, 13, 16
