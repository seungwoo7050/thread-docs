# 성능 측정·격리 테스트·복구·릴리스 관문

부하 생성기의 시간 불변식, disposable 환경 검증, live source-to-SSR 검증, 파일 시스템 TOCTOU 방어, 복구와 배포 관문을 다룬다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P28. [Thread 20 / `fix(perf): recover paced schedule without bursts`] 지연을 따라잡되 burst를 만들지 않는 paced scheduler

**우선순위:** A

### 면접 질문

- fixed-rate 부하 생성기에서 작업이 늦어졌을 때 원래 시각을 한꺼번에 따라잡으면 어떤 측정 왜곡이 생기나요?
- nominal interval과 minimum interval을 함께 두는 이유는 무엇인가요?
- `time.time()` 대신 monotonic clock을 사용해야 하는 이유는 무엇인가요?
- 꼬리 질문: planned jitter와 실제 submit 간격을 각각 측정하면 무엇을 구분할 수 있나요?
  - **모범답변:** **프로젝트 특수사항:** planned jitter는 paced deadline과 worker 시작의 차이이고 실제 submit 간격은 producer 제출 시각 사이의 차이라서 queue 지연과 catch-up burst를 구분합니다. 최종 `latency_ms`는 HTTP 구간만이 아니라 paced deadline부터 완료까지의 end-to-end 값으로, jitter와 service latency를 함께 포함합니다. **일반 원칙:** 일정 지연을 별도 지표로 남겨야 end-to-end latency 증가 중 부하 생성기 지연의 몫을 해석할 수 있습니다.

### 30초 모범 답변

fixed-rate 일정이 밀렸다고 즉시 여러 요청을 제출하면 실제 시스템 부하가 아닌 scheduler의 catch-up burst를 측정하게 됩니다. 그래서 정상 상태에서는 nominal 간격을 따르되 지연 후에도 직전 제출 시각 기준 minimum 간격을 지키며 점진적으로 복귀합니다. 시간 계산은 wall clock 보정의 영향을 받지 않는 monotonic clock을 쓰고, 계획 대비 지연과 실제 제출 간격을 따로 기록해 scheduler 오차와 서버 지연을 구분합니다.

### 답변 핵심 키워드

`monotonic clock`, `fixed-rate`, `catch-up burst`, `minimum interval`, `gradual recovery`, `planned jitter`, `submission interval`

### 백지 구현

**구현 목표**

nominal interval을 목표로 하되 이전 제출과의 minimum interval을 위반하지 않는 O(1) paced scheduler 상태 객체를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
class Pacer:
    def __init__(
        self,
        *,
        interval_seconds: float,
        minimum_interval_seconds: float,
        started_at: float,
    ) -> None:
        if (
            not math.isfinite(interval_seconds)
            or not math.isfinite(minimum_interval_seconds)
            or not math.isfinite(started_at)
            or interval_seconds <= 0
            or minimum_interval_seconds <= 0
            or minimum_interval_seconds > interval_seconds
        ):
            raise PacerError("pacer_configuration_invalid")
        self._interval = interval_seconds
        self._minimum_interval = minimum_interval_seconds
        self._next_nominal_at = started_at
        self._last_submitted_at: float | None = None
        self._last_observed_at = started_at
        self._pending_deadline: float | None = None

    def next_delay(self, now: float) -> float:
        if not math.isfinite(now) or now < self._last_observed_at:
            raise PacerError("monotonic_time_reversed")
        self._last_observed_at = now

        deadline = self._next_nominal_at
        if self._last_submitted_at is not None:
            # 지연된 nominal slot을 복구하더라도 직전 실제 submit 기준 floor는 지킨다.
            deadline = max(deadline, self._last_submitted_at + self._minimum_interval)
        self._pending_deadline = deadline
        return max(0.0, deadline - now)

    def mark_submitted(self, submitted_at: float) -> None:
        if (
            not math.isfinite(submitted_at)
            or submitted_at < self._last_observed_at
            or self._pending_deadline is None
            or submitted_at + 1e-12 < self._pending_deadline
            or (
                self._last_submitted_at is not None
                and submitted_at <= self._last_submitted_at
            )
        ):
            raise PacerError("submission_state_invalid")
        self._last_observed_at = submitted_at
        self._last_submitted_at = submitted_at
        self._next_nominal_at += self._interval
        self._pending_deadline = None
```

**입력과 출력**

- 입력: monotonic 기준 현재 시각과 실제 제출 시각
- 출력: 다음 제출 전 대기해야 할 0 이상의 초

**반드시 만족해야 할 조건**

- 정상 경로에서는 nominal interval을 유지한다.
- 일정이 밀려도 연속 제출 간격이 minimum interval보다 짧아지지 않는다.
- 지연을 여러 번의 0초 대기로 한꺼번에 따라잡지 않는다.
- 제출이 완료된 시점만 다음 간격 계산에 반영한다.
- interval과 minimum interval의 관계를 생성 시 검증한다.
- 모든 계산은 monotonic timestamp를 전제로 한다.

**경계 조건**

- 첫 제출 전
- 정확히 nominal 시각에 도달한 경우
- 아주 긴 일시 정지 뒤 재개
- floating-point 오차로 계산값이 아주 작은 음수가 되는 경우
- minimum interval과 nominal interval이 같은 경우

**실패 조건**

- 0 이하 interval
- minimum interval이 nominal interval보다 큰 설정
- 시각이 역행하는 입력
- 같은 제출을 중복 기록하는 상태

**제약**

- sleep이나 thread 생성 자체는 구현하지 않는다.
- 내부 상태와 산술만 구현한다.
- 호출당 시간·공간 복잡도는 O(1)이다.
- 15~20분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 정상 상태에서 제출 간격이 nominal interval이다.
- [ ] 긴 정지 직후에도 0초 제출이 연속으로 나오지 않는다.
- [ ] 모든 실제 제출 간격이 minimum interval 이상이다.
- [ ] wall clock 변경을 가정한 API가 없다.
- [ ] 부동소수점 오차 때문에 음수 sleep이 반환되지 않는다.
- [ ] 호출당 상태 크기가 요청 수에 따라 늘어나지 않는다.

### 구현 후 설명할 것

- fixed-rate와 fixed-delay의 차이
  - **모범답변:** fixed-rate는 시작 시각과 index로 nominal deadline을 계산해 장기 목표 rate를 유지하고, fixed-delay는 직전 작업 완료 시각부터 매번 같은 간격을 둡니다. 원본 부하 도구는 fixed-rate deadline을 쓰되 실제 submit 뒤 최소 복구 간격을 함께 적용합니다.
- catch-up burst를 금지한 이유와 부하 정확도 trade-off
  - **모범답변:** 밀린 slot을 즉시 연속 제출하면 서버가 아니라 generator가 만든 순간 부하를 측정하게 됩니다. 최소 간격은 그 왜곡을 막지만, 큰 지연 뒤에는 계획한 누적 요청 시각을 즉시 회복하지 못하므로 report에 schedule jitter와 throughput을 함께 남겨야 합니다.
- planned jitter와 실제 request latency를 분리하는 방법
  - **모범답변:** 각 요청에 paced deadline을 전달해 worker 시작과의 차이를 schedule jitter로 별도 기록합니다. 원본의 최종 latency는 `max(완료 시각 - paced deadline, schedule jitter + requester latency)`로 재구성한 end-to-end 값이며, submit timestamp 쌍은 실제 최소 간격 검증에 씁니다. 따라서 jitter를 latency와 나란히 봐야 generator 대기와 요청 처리의 영향을 해석할 수 있습니다.
- monotonic clock이 필요한 이유
  - **모범답변:** wall clock은 NTP 보정이나 운영자 변경으로 앞뒤로 이동할 수 있어 음수 지연이나 잘못된 burst 판정을 만들 수 있습니다. 경과 시간과 deadline 계산에는 역행하지 않는 monotonic clock을 사용하고, 사람이 읽는 시각이 필요할 때만 wall clock을 별도로 사용합니다.
- 최소 간격을 두면서 목표 rate로 복귀하는 정책
  - **모범답변:** 다음 deadline을 `max(nominal_deadline, previous_submission + floor)`로 정합니다. 정상일 때는 nominal rate를 따르고, 오래 멈춘 뒤에는 한 건만 즉시 제출한 다음 floor 간격으로 따라잡으므로 O(1) 상태로 burst 없이 점진 복귀합니다.

### 원본 확인 위치

- Thread 20
- 커밋 `fix(perf): measure paced schedule jitter`
- 커밋 `fix(perf): recover paced schedule without bursts`
- 파일 `scripts/http_load_profile.py`
- 구성 요소 `run_profile`, `RunMeasurements`, `LoadReport`, `_ActiveRequestCounter`

## P29. [Thread 21 / `test(history): build vnext browser fixture`] 브라우저 fixture를 disposable 환경에만 허용하는 fail-closed gate

**우선순위:** A

### 면접 질문

- 실제 publication lifecycle을 재현하는 브라우저 fixture를 일반 개발 DB나 production-like 환경에서 실행하면 왜 위험한가요?
- 단순히 `DEBUG=True`만 확인하는 것으로 충분하지 않은 이유는 무엇인가요?
- fixture에서 외부 source client 호출을 명시적으로 금지한 이유는 무엇인가요?
- 꼬리 질문: 환경 검증과 데이터 생성 사이의 TOCTOU를 어떻게 줄일 수 있나요?
  - **모범답변:** **프로젝트 특수사항:** `transaction.atomic`인 fixture 함수의 첫 단계에서 설정·DB identity·root row 부재를 확인하고 같은 연결로 곧바로 생성합니다. **일반 원칙:** 전용 최소 권한 계정과 새 disposable DB를 쓰고, 같은 transaction에서 잠금 또는 쓰기 직전 재검사로 동시 writer의 경쟁 창을 줄입니다.

### 30초 모범 답변

fixture는 review·seal·activation과 대량 typed fact를 실제 DB에 만들기 때문에 전용 disposable DB에서만 허용해야 합니다. `DEBUG` 외에도 QA opt-in, Admin·control plane 비활성, loopback의 전용 DB 이름, 초기 empty 상태를 함께 확인해 잘못된 대상에서 실패 폐쇄합니다. 외부 source 호출은 mock으로 금지하고, fixture 자체가 생성한 결정적 데이터로 실제 service 경로를 통과시켜 UI 검증과 source 테스트의 책임을 분리합니다.

### 답변 핵심 키워드

`disposable database`, `fail closed`, `explicit opt-in`, `loopback`, `empty precondition`, `source call forbidden`, `real lifecycle service`

### 백지 구현

**구현 목표**

브라우저 fixture 실행 전에 환경 설정과 DB 상태를 검증하는 순수 validation 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def validate_fixture_environment(
    *,
    debug: bool,
    qa_enabled: bool,
    admin_enabled: bool,
    control_plane_enabled: bool,
    database: Mapping[str, object],
    occupied: bool,
) -> None:
    if (
        debug is not True
        or qa_enabled is not True
        or admin_enabled is not False
        or control_plane_enabled is not False
    ):
        raise FixtureEnvironmentError("qa_fixture_environment_denied")

    name = database.get("NAME")
    host = database.get("HOST")
    port = database.get("PORT")
    if (
        database.get("ENGINE") != "django.db.backends.postgresql"
        or host not in {"127.0.0.1", "localhost", "::1"}
        or port not in {55434, "55434"}
        or type(name) is not str
        or re.fullmatch(r"grocery_vnext_[a-z0-9_]+", name) is None
    ):
        raise FixtureEnvironmentError("qa_fixture_database_denied")
    if occupied:
        raise FixtureEnvironmentError("qa_fixture_database_not_empty")
```

**입력과 출력**

- 입력: runtime flag, DB engine·name·host·port, DB 점유 여부
- 출력: 없음. 안전 계약을 만족하지 않으면 고정 오류 코드의 예외

**반드시 만족해야 할 조건**

- 명시적 QA opt-in과 debug가 모두 켜져 있어야 한다.
- Admin과 production control plane은 꺼져 있어야 한다.
- PostgreSQL의 승인된 loopback host·전용 port·전용 DB naming contract만 허용한다.
- 일반 application DB 이름을 거부한다.
- 기존 root 데이터가 하나라도 있으면 거부한다.
- 오류 메시지에 supplied DB 이름·host 같은 원문을 반사하지 않는다.

**경계 조건**

- `localhost`와 숫자 loopback 표현 정책
- 누락된 setting
- boolean처럼 보이는 문자열
- empty schema지만 migration만 존재하는 경우의 점유 판정
- fixture 전용 이름과 비슷한 prefix만 가진 이름

**실패 조건**

- production 또는 shared DB 가능성이 있는 설정
- QA opt-in 누락
- Admin·control plane 활성
- 원격 host
- 이미 데이터가 있는 DB
- 오류에 connection 정보 노출

**제약**

- 실제 DB 연결이나 mutation은 하지 않는다.
- allowlist 방식으로 검증한다.
- 15분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 안전한 한 조합만 통과한다.
- [ ] flag 하나씩 반전한 모든 조합이 거부된다.
- [ ] 일반 DB 이름과 원격 host가 거부된다.
- [ ] occupied 상태가 거부된다.
- [ ] 오류 문자열에 입력 marker가 포함되지 않는다.
- [ ] validation 성공 전에는 어떠한 write도 일어나지 않는 구조다.

### 구현 후 설명할 것

- `DEBUG=True` 하나만으로 안전 경계가 되지 않는 이유
  - **모범답변:** DEBUG는 오류 표시 동작일 뿐 DB가 disposable인지, Admin이 꺼졌는지, fixture 실행을 명시적으로 허용했는지를 증명하지 않습니다. 프로젝트는 QA opt-in, Admin 비활성, PostgreSQL loopback·전용 이름, root row 부재를 함께 검사합니다.
- denylist보다 allowlist가 적합한 이유
  - **모범답변:** 위험한 host나 DB 이름을 열거하면 새 환경과 별칭을 놓칠 수 있습니다. 통과 가능한 engine, loopback host, 전용 port·이름 prefix, 정확한 boolean 조합 하나만 허용하면 모르는 설정은 기본적으로 거부됩니다.
- environment gate와 DB 권한 분리
  - **모범답변:** application validation은 실수 방지용 방어층이지 보안 권한 자체가 아닙니다. 별도의 disposable DB와 최소 권한 계정으로 production·shared DB에 연결하거나 변경할 능력 자체를 제한해야 설정 검증 우회에도 피해가 bounded됩니다.
- 외부 source 호출을 fixture에서 차단하는 테스트 설계
  - **모범답변:** fixture는 결정적 합성 fact를 만들고 실제 review·seal·activation service를 통과시킵니다. source client 메서드는 예외를 내도록 patch해 브라우저 검증이 네트워크·credential·provider 가용성에 우연히 의존하는 회귀를 즉시 잡습니다.
- 검증 후 write까지의 경쟁 조건을 줄이는 방법
  - **모범답변:** 환경 검증을 첫 write 직전에 같은 connection·transaction에서 수행하고, 가능하면 root table lock이나 advisory lock으로 경쟁 fixture를 직렬화합니다. 전용으로 새로 만든 DB와 단일 실행 계정을 쓰는 것이 더 강한 외부 경계입니다.

### 원본 확인 위치

- Thread 21
- 커밋 `test(history): build vnext browser fixture`
- 커밋 `fix(qa): require disposable browser database`
- 파일 `grocery/tests/vnext_browser_fixture.py`
- 구성 요소 `build_vnext_browser_fixture`
- 연관 Thread 18

## P30. [Thread 21 / `test(source): add guarded live source-to-SSR smoke`] live source를 한 번만 통과시키고 SSR 경계는 격리하는 E2E 설계

**우선순위:** A

### 면접 질문

- live API E2E에서 source 호출과 SSR request를 같은 단계에 섞지 않은 이유는 무엇인가요?
- `CachedLiveClient`와 이후 source call 금지가 어떤 결함을 잡아내나요?
- 테스트가 실패하더라도 disposable DB의 root row가 남지 않았음을 확인해야 하는 이유는 무엇인가요?
- 꼬리 질문: live 테스트 receipt와 failure code를 어떻게 설계해야 secret이나 provider 원문이 노출되지 않나요?
  - **모범답변:** **프로젝트 특수사항:** 성공 receipt는 네 종류의 row count와 고정 상태만 담고, 실패는 stage와 정규식으로 제한된 code만 남깁니다. **일반 원칙:** provider 예외 본문이나 query를 `str(error)`로 출력하지 말고 승인 code가 없으면 안전한 타입명 또는 `internal_error`로 축약합니다.

### 30초 모범 답변

live 단계는 명시적 opt-in 환경에서 bounded source 응답을 한 번 획득하고, 이후 ingestion·review·publication·SSR은 그 캡처된 결과만 사용합니다. SSR 중 source 호출을 금지하면 공개 request가 외부 네트워크나 credential에 의존하는 회귀를 잡을 수 있습니다. 전체 흐름은 disposable DB transaction·정리 절차와 rollback 검증을 갖고, 성공 receipt와 실패는 allowlisted 식별자·고정 코드만 출력합니다.

### 답변 핵심 키워드

`live boundary`, `cached response`, `SSR isolation`, `explicit opt-in`, `bounded acquisition`, `rollback verification`, `safe receipt`, `fixed failure code`

### 백지 구현

**구현 목표**

환경 검증, 단일 live fetch, 캐시된 pipeline 실행, SSR source-call 금지, cleanup 검증을 단계 상태로 관리하는 orchestration 함수를 설계한다.

**인터페이스 또는 함수 시그니처**

```python
def run_live_smoke(
    *,
    environment: LiveSmokeEnvironment,
    live_client: LiveSourceClient,
    pipeline: PublicationPipeline,
    public_probe: PublicProbe,
    cleanup: CleanupBoundary,
) -> LiveSmokeReceipt:
    # 원본처럼 opt-in·loopback·전용 DB·empty root 검증이 어떤 I/O보다 먼저다.
    try:
        environment.validate()
    except Exception as error:
        raise LiveSmokeFailure("ENVIRONMENT", safe_failure_code(error)) from None

    primary_failure: LiveSmokeFailure | None = None
    cleanup_failure: LiveSmokeFailure | None = None
    receipt: LiveSmokeReceipt | None = None
    stage = "ACQUISITION"
    try:
        budget = environment.acquisition_budget
        captured = live_client.fetch(
            max_pages=budget.max_pages,
            max_rows=budget.max_rows,
            max_page_bytes=budget.max_page_bytes,
            timeout_seconds=budget.timeout_seconds,
        )
        if not captured.rows:
            raise LiveSmokeInvariantError("live_dataset_empty")

        # raw transport 대신 immutable cached result만 application 단계에 넘긴다.
        cached = pipeline.freeze_capture(captured)
        stage = "INGESTION"
        outcome = pipeline.ingest(cached)
        stage = "REVIEW"
        decision = pipeline.review(outcome)
        stage = "PUBLICATION"
        revision = pipeline.seal(decision)
        pipeline.activate(revision)

        stage = "SSR"
        # 원본은 이 구간의 source client 메서드를 fixed error로 patch한다.
        with live_client.forbid_calls():
            public_probe.verify(revision)
        receipt = LiveSmokeReceipt(
            recent_rows=outcome.recent_rows,
            monthly_rows=outcome.monthly_rows,
            regional_rows=outcome.regional_rows,
            market_rows=outcome.market_rows,
        )
    except Exception as error:
        primary_failure = LiveSmokeFailure(stage, safe_failure_code(error))
    finally:
        try:
            cleanup.rollback()
        except Exception as error:
            cleanup_failure = LiveSmokeFailure("CLEANUP", safe_failure_code(error))
        try:
            # rollback 시도 자체가 실패해도 잔존 row 검증은 별도로 수행한다.
            cleanup.require_no_root_rows()
        except Exception as error:
            cleanup_failure = LiveSmokeFailure("CLEANUP", safe_failure_code(error))

    if primary_failure is not None and cleanup_failure is not None:
        # 두 raw 예외 대신 stage/code만 가진 독립 실패를 함께 보존한다.
        raise ExceptionGroup("live_smoke_failed", [primary_failure, cleanup_failure])
    if primary_failure is not None:
        raise primary_failure
    if cleanup_failure is not None:
        raise cleanup_failure
    if receipt is None:
        raise LiveSmokeFailure("RECEIPT", "receipt_missing")
    return receipt
```

**입력과 출력**

- 입력: 검증 가능한 환경, live client, publication pipeline, SSR probe, cleanup 경계
- 출력: 고정 필드만 가진 성공 receipt

**반드시 만족해야 할 조건**

- 환경 validation이 가장 먼저 실행된다.
- source fetch에는 page·row·byte·timeout 상한이 있다.
- live response는 이후 단계에서 재사용 가능한 캐시 형태로 고정한다.
- ingestion 이후 SSR probe에서는 live client 호출을 거부한다.
- review·seal·activate를 실제 service boundary로 수행한다.
- 어느 단계에서 실패해도 cleanup을 시도하고 최종 root row 부재를 검증한다.
- 원래 실패와 cleanup 실패를 구분하되 private 원문을 출력하지 않는다.

**경계 조건**

- live response가 0행인 경우
- 일부 historical source만 실패한 경우
- publication까지 성공한 뒤 SSR 검증 실패
- cleanup 자체 실패
- 같은 cached response의 재사용

**실패 조건**

- opt-in·환경 불일치
- acquisition budget 초과
- SSR 중 source 호출
- lifecycle invariant 불일치
- rollback 후 row 잔존
- receipt에 secret·query·provider body 노출

**제약**

- 실제 HTTP 구현은 인터페이스 뒤에 둔다.
- orchestration과 validation을 분리한다.
- 25~30분 이내에 핵심 상태 흐름만 구현한다.

### 구현 후 자가 검증

- [ ] 환경 검증 실패 시 live client가 호출되지 않는다.
- [ ] live client 호출 횟수가 설계한 상한을 넘지 않는다.
- [ ] SSR probe에서 source 호출을 시도하면 즉시 실패한다.
- [ ] 각 단계 실패 후 cleanup이 실행된다.
- [ ] cleanup 후 root row 부재를 확인한다.
- [ ] receipt와 예외에 private marker가 없다.
- [ ] acquisition 실패와 cleanup 실패가 구분된다.

### 구현 후 설명할 것

- live source 검증과 deterministic application 검증을 분리한 이유
  - **모범답변:** live 단계는 provider 계약·credential·budget을 검증하고, 이후 단계는 캡처된 결과로 ingestion부터 SSR까지를 검증합니다. 이 경계를 분리해야 application 실패를 provider 변동으로 오인하지 않고, SSR이 외부 네트워크를 호출하는 회귀도 잡을 수 있습니다.
- 캐시된 응답을 경계 객체로 만든 trade-off
  - **모범답변:** 원본 `CachedLiveClient`는 dataset·query·page size·한 번 사용 조건을 검사해 정상 persistence workflow에 동일 결과를 한 번만 공급합니다. 재현성은 좋아지지만 live transport의 모든 재시도 조합을 뒤 단계에서 다시 시험하지 않으므로 acquisition 단계의 별도 receipt가 필요합니다.
- cleanup을 best-effort로만 두지 않고 사후 검증한 이유
  - **모범답변:** rollback 호출이 성공해 보여도 잘못된 transaction 경계나 별도 connection write로 root row가 남을 수 있습니다. 원본은 atomic block을 rollback한 뒤 root model 부재를 다시 조회해 disposable 상태 복원을 결과 invariant로 검증합니다.
- 단계별 고정 failure code 설계
  - **모범답변:** `ENVIRONMENT`, `INGESTION`, `SSR`, `ROLLBACK` 같은 bounded stage와 allowlisted code를 조합하면 운영자는 실패 위치를 알 수 있지만 provider body·query·secret은 보지 않습니다. 승인 code가 아니면 타입명이나 `internal_error`로 축약합니다.
- production worker나 일반 CI에서 이 smoke를 금지하는 방법
  - **모범답변:** 명시적 환경 opt-in, DEBUG·기능 flag 조합, exact disposable DB identity와 empty 검사를 모두 요구하고 일반 `check`·`production-check` target에는 연결하지 않습니다. 여기에 live credential 접근 권한과 네트워크 egress를 전용 job에만 부여합니다.

### 원본 확인 위치

- Thread 21
- 커밋 `test(source): add guarded live source-to-SSR smoke`
- 파일 `scripts/live_api_e2e_smoke.py`
- 구성 요소 `validate_disposable_environment`, `CachedLiveClient`, `LiveSmokeInvariantError`, `LiveSmokeReceipt`, `safe_failure_code`
- 연관 Thread 03, 05, 11, 13

## P31. [Thread 22 / `fix(ops): harden postgres recovery boundaries`] path 검증을 file descriptor identity로 고정하는 TOCTOU 방어

**우선순위:** S

### 면접 질문

- backup 파일의 path를 검사한 뒤 나중에 같은 path를 다시 열면 어떤 TOCTOU 공격이나 사고가 가능한가요?
- `O_NOFOLLOW`, `lstat`, `fstat`, owner·mode·regular-file 검증은 각각 무엇을 막나요?
- checksum 검증과 실제 restore가 같은 열린 descriptor를 사용해야 하는 이유는 무엇인가요?
- 꼬리 질문: descriptor identity를 중간 단계마다 다시 확인하는 이유와 한계는 무엇인가요?
  - **모범답변:** **프로젝트 특수사항:** device·inode·size·mtime·ctime을 고정하고 단계 사이 `fstat`으로 비교해 열린 파일의 수정·교체 흔적을 탐지합니다. **일반 원칙:** 이 검사는 검사 순간 사이의 write, hard link 변경, 상위 경로 정책까지 완전히 막지 못하므로 owner-only directory·권한과 immutable backup 정책을 함께 적용합니다.

### 30초 모범 답변

path는 검사 이후 rename이나 symlink 교체로 다른 객체를 가리킬 수 있으므로, 안전하게 연 파일 descriptor를 신뢰 단위로 사용합니다. `O_NOFOLLOW`로 마지막 symlink를 거부하고 `fstat`으로 regular file·owner·private mode·size를 검증한 뒤 device/inode 같은 identity를 고정합니다. checksum, magic 검사, restore까지 같은 descriptor를 전달하고 단계 사이 identity를 재확인하며, 모든 예외 경로에서 descriptor를 닫습니다.

### 답변 핵심 키워드

`TOCTOU`, `O_NOFOLLOW`, `lstat/fstat`, `regular file`, `owner-controlled`, `private mode`, `descriptor identity`, `same FD`, `resource cleanup`

### 백지 구현

**구현 목표**

symlink·비정규 파일·권한 확장·크기 초과를 거부하고, 검증된 하나의 descriptor와 identity를 context manager로 반환한다.

**인터페이스 또는 함수 시그니처**

```python
@contextmanager
def open_verified_regular_file(
    path: Path,
    *,
    maximum_bytes: int,
) -> Iterator[VerifiedFile]:
    if type(maximum_bytes) is not int or maximum_bytes < 1:
        raise VerifiedFileError("maximum_size_invalid")

    try:
        before = path.lstat()
    except OSError:
        raise VerifiedFileError("file_open_denied") from None
    if stat.S_ISLNK(before.st_mode) or not stat.S_ISREG(before.st_mode):
        raise VerifiedFileError("file_type_invalid")

    flags = os.O_RDONLY | os.O_CLOEXEC | os.O_NOFOLLOW
    try:
        descriptor = os.open(path, flags)
    except OSError:
        raise VerifiedFileError("file_open_denied") from None

    try:
        try:
            metadata = os.fstat(descriptor)
        except OSError:
            raise VerifiedFileError("file_metadata_invalid") from None
        if (
            not stat.S_ISREG(metadata.st_mode)
            or (metadata.st_dev, metadata.st_ino) != (before.st_dev, before.st_ino)
            or metadata.st_uid != os.geteuid()
            or stat.S_IMODE(metadata.st_mode) & 0o077
        ):
            raise VerifiedFileError("file_permissions_invalid")
        if metadata.st_size < 1 or metadata.st_size > maximum_bytes:
            raise VerifiedFileError("file_size_invalid")

        identity = (
            metadata.st_dev,
            metadata.st_ino,
            metadata.st_size,
            metadata.st_mtime_ns,
            metadata.st_ctime_ns,
        )
        # descriptor의 ownership은 context manager에 남고 호출자는 빌려 쓴다.
        yield VerifiedFile(
            descriptor=descriptor,
            identity=identity,
            size=metadata.st_size,
        )
    finally:
        try:
            os.close(descriptor)
        except OSError:
            raise VerifiedFileError("file_close_failed") from None
```

**입력과 출력**

- 입력: 읽기 전용으로 열 backup member 경로와 최대 크기
- 출력: 열린 descriptor, 검증 시 identity, 크기를 가진 `VerifiedFile`

**반드시 만족해야 할 조건**

- 마지막 경로 요소의 symlink follow를 금지한다.
- 열린 뒤 descriptor 기준 regular file인지 확인한다.
- 현재 operator 또는 승인된 owner 정책을 만족해야 한다.
- group/world permission이 허용 범위를 넘으면 거부한다.
- 파일 크기가 1 이상 maximum 이하인지 확인한다.
- 검증 후 path를 다시 열지 않고 동일 descriptor를 호출자에게 제공한다.
- 성공·실패·호출자 예외 모든 경로에서 descriptor를 정확히 한 번 닫는다.
- 검증 오류는 고정 코드로 변환하고 path나 OS 원문을 반사하지 않는다.

**경계 조건**

- 파일이 open 직전 사라지는 경우
- symlink와 hard link 정책의 차이
- 0바이트와 정확히 maximum 크기
- 파일이 열린 뒤 rename되는 경우
- descriptor를 duplicate해 읽는 경우

**실패 조건**

- symlink·directory·device·socket
- owner 불일치
- broad permissions
- 크기 초과
- open/fstat/read 오류
- descriptor leak 또는 이중 close

**제약**

- POSIX 환경을 전제로 한다.
- checksum이나 PostgreSQL restore 자체는 구현하지 않는다.
- context manager와 검증 경계만 20~25분 내 구현한다.

### 구현 후 자가 검증

- [ ] symlink 입력이 거부된다.
- [ ] directory와 비정규 파일이 거부된다.
- [ ] broad mode가 거부된다.
- [ ] 최대 크기 경계가 정확하다.
- [ ] path가 rename되어도 열린 descriptor는 원래 파일을 읽는다.
- [ ] 모든 예외 경로에서 descriptor가 닫힌다.
- [ ] 검증 후 같은 path를 재개방하지 않는다.
- [ ] 오류에 path·credential·OS 원문이 없다.

### 구현 후 설명할 것

- path identity와 open file description의 차이
  - **모범답변:** path는 디렉터리 entry를 다시 해석할 때마다 다른 inode를 가리킬 수 있지만, 열린 descriptor는 open 당시의 file description과 inode를 계속 가리킵니다. rename 뒤에도 같은 descriptor로 읽으면 검증한 객체를 유지할 수 있습니다.
- `lstat` 사전 검사만으로 충분하지 않은 이유
  - **모범답변:** `lstat(path)`와 `open(path)` 사이에 공격자나 다른 process가 path를 바꿀 수 있습니다. 따라서 사전 검사는 보조 신호일 뿐이고, `O_NOFOLLOW`로 연 뒤 `fstat(descriptor)` 결과를 신뢰해야 합니다.
- `O_NOFOLLOW`가 막는 범위와 hard link 같은 남은 위험
  - **모범답변:** `O_NOFOLLOW`는 마지막 경로 요소가 symlink인 경우 follow를 막지만 상위 경로 교체나 동일 inode의 hard link, owner가 가진 기존 file 수정까지 막지는 않습니다. trusted owner-only directory descriptor와 private mode, 운영상 불변 파일 정책이 추가로 필요합니다.
- 동일 descriptor를 checksum·validation·restore에 재사용한 이유
  - **모범답변:** path를 다시 열면 checksum한 파일과 restore한 파일이 달라질 수 있습니다. 원본은 같은 descriptor를 rewind해 magic, bounded read, SHA-256을 계산하고 restore 입력에도 전달하며 단계 사이 identity를 재확인합니다.
- resource cleanup과 error redaction 설계
  - **모범답변:** context manager가 descriptor의 유일한 owner가 되어 validation 실패, 호출자 예외, 성공 모두에서 한 번 닫습니다. `OSError` 원문과 path는 밖으로 내보내지 않고 `file_open_denied` 같은 고정 code로 바꿔 filesystem 정보 노출을 막습니다.

### 원본 확인 위치

- Thread 22
- 커밋 `feat(ops): rehearse postgres recovery`
- 커밋 `fix(ops): harden postgres recovery boundaries`
- 파일 `scripts/postgres_backup_restore.py`
- 구성 요소 `_open_private_regular_file`, `_require_private_regular_descriptor`, `_file_identity`, `_require_descriptor_identity`, `_read_bounded_descriptor`, `_read_prefix_descriptor`, `_file_sha256_descriptor`

## P32. [Thread 22 / `docs: record predeploy completion evidence`] application·publication·database rollback을 분리한 릴리스 관문

**우선순위:** A

### 면접 질문

- application rollback, publication rollback, database recovery를 하나의 “롤백”으로 취급하면 왜 위험한가요?
- exact release SHA, clean Git, forward migration plan, secret/dependency/license gate를 배포 전에 묶은 이유는 무엇인가요?
- backup 파일의 checksum만 맞는 것으로 복구 가능성을 증명할 수 없는 이유는 무엇인가요?
- 꼬리 질문: migration을 역방향으로 실행하지 않고 이전 application을 최신 schema에 맞춰 검증하는 전략의 trade-off는 무엇인가요?
  - **모범답변:** **프로젝트 특수사항:** code rollback 때 production migration을 역방향으로 실행하지 않는 정책이지만, 과거 Phase 0 rehearsal은 `0010`까지만의 증거이고 exact 이전 release의 `0028` schema·vNext route 호환성은 아직 미검증입니다. 따라서 별도 rehearsal 전에는 그 SHA로 traffic을 전환하지 않습니다. **일반 원칙:** DB 역변경 위험은 줄지만 expand-and-contract와 구 code 호환성 검증이 필요하고 파괴적 정리를 늦춰 migration 복잡도와 저장 비용이 늘어납니다.

### 30초 모범 답변

세 rollback은 대상과 위험이 다릅니다. application은 이전 immutable artifact로 traffic을 전환하고, publication은 append-only activation으로 이전 검수본을 가리키며, database는 기존 DB를 덮지 않고 격리 restore나 PITR로 새 대상을 검증합니다. 배포 관문은 exact SHA·clean tree·production 설정·secret·dependency·license·forward migration·health를 같은 evidence로 묶고, restore 후 row count·migration inventory·active revision·canonical publication hash까지 대사합니다.

### 답변 핵심 키워드

`separate rollback domains`, `immutable release`, `exact SHA`, `forward-only migration`, `restore rehearsal`, `out-of-band manifest`, `canonical publication`, `fail closed`

### 백지 구현

**구현 목표**

여러 검증 결과를 받아 모든 필수 증거가 일관될 때만 승인 receipt를 만드는 release gate evaluator를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def evaluate_release_gate(
    candidate: ReleaseCandidate,
    evidence: ReleaseEvidence,
) -> ReleaseGateReceipt:
    sha = candidate.release_sha
    if type(sha) is not str or re.fullmatch(r"[0-9a-f]{40}", sha) is None:
        raise ReleaseGateError("release_sha_invalid")
    if not evidence.release_shas or any(value != sha for value in evidence.release_shas):
        raise ReleaseGateError("release_sha_mismatch")

    required_gates = {
        "git_clean": evidence.git_clean,
        "locked_install": evidence.locked_install_ok,
        "secret_scan": evidence.secret_scan_ok,
        "dependency_audit": evidence.dependency_audit_ok,
        "license_inventory": evidence.license_inventory_ok,
        "static_build": evidence.static_build_ok,
        "migration_plan": evidence.forward_migration_plan_ok,
        "migration_check": evidence.migration_check_ok,
        "schema_compatibility": evidence.previous_application_compatible,
    }
    if any(value is not True for value in required_gates.values()):
        raise ReleaseGateError("required_gate_failed")

    settings_state = evidence.production_settings
    if (
        settings_state.debug is not False
        or settings_state.admin_enabled is not False
        or settings_state.qa_enabled is not False
        or settings_state.control_plane_enabled is not False
        or settings_state.validated is not True
    ):
        raise ReleaseGateError("production_settings_invalid")
    if candidate.publication_state == "WITHDRAWN":
        raise ReleaseGateError("publication_withdrawn")

    restore = evidence.restore
    sha256_values = (
        restore.backup_sha256,
        restore.manifest_dump_sha256,
        restore.manifest_sha256,
        restore.expected_manifest_sha256,
        restore.fact_set_sha256,
        candidate.expected_fact_set_sha256,
    )
    if (
        any(
            type(value) is not str
            or re.fullmatch(r"[0-9a-f]{64}", value) is None
            for value in sha256_values
        )
        # manifest bytes는 독립 receipt와, dump bytes는 manifest 내부 digest와 각각 대사한다.
        or restore.manifest_sha256 != restore.expected_manifest_sha256
        or restore.backup_sha256 != restore.manifest_dump_sha256
        or restore.manifest_is_out_of_band is not True
        or restore.isolated_restore_succeeded is not True
        or restore.source_database_untouched is not True
        or restore.row_counts != candidate.expected_row_counts
        or restore.migration_inventory != candidate.expected_migration_inventory
        or restore.current_publication != candidate.expected_current_publication
        or restore.fact_set_sha256 != candidate.expected_fact_set_sha256
    ):
        raise ReleaseGateError("restore_reconciliation_failed")

    rollback = evidence.rollback
    if rollback.application_ready is not True:
        raise ReleaseGateError("application_rollback_unready")
    if rollback.publication_ready is not True:
        raise ReleaseGateError("publication_rollback_unready")
    if rollback.database_recovery_ready is not True:
        raise ReleaseGateError("database_recovery_unready")

    # receipt는 승인된 bounded 식별자와 gate code만 포함한다.
    return ReleaseGateReceipt(
        status="APPROVED",
        release_sha=sha,
        passed_gates=tuple(required_gates),
        restored_fact_set_sha256=restore.fact_set_sha256,
    )
```

**입력과 출력**

- 입력: 기대 release SHA·migration 범위·publication 계약과 각 gate의 구조화 evidence
- 출력: allowlisted 필드만 포함한 승인 receipt

**반드시 만족해야 할 조건**

- release SHA는 exact lowercase 40자리이고 모든 evidence와 일치해야 한다.
- Git tree가 clean하고 lock 기반 설치·검사가 성공해야 한다.
- production 설정은 debug·Admin·QA·control-plane 노출을 fail-closed로 검증한다.
- secret scan, dependency audit, license inventory, static build가 모두 통과해야 한다.
- migration은 forward plan·check 결과와 호환성 evidence가 있어야 한다.
- backup은 out-of-band manifest digest와 일치하고 격리 restore가 성공해야 한다.
- restore 후 row counts, migration inventory, current publication, canonical fact-set hash가 일치해야 한다.
- application·publication·database rollback evidence를 서로 대체하지 않는다.
- 실패 receipt는 고정 코드만 사용하고 supplied secret·path·SHA 일부를 임의로 반사하지 않는다.

**경계 조건**

- 모든 검사 성공이지만 release SHA 하나만 불일치
- backup hash는 맞지만 publication contract 불일치
- application rollback은 가능하지만 이전 code가 새 schema와 비호환
- publication이 withdrawn인 candidate
- cleanup이 필요한 실패한 restore target

**실패 조건**

- gate 누락 또는 evidence 중복·충돌
- dirty tree
- production setting drift
- secret 또는 dependency/license 검사 실패
- reverse/fake migration에 의존
- source DB overwrite를 전제로 한 restore
- 서로 다른 rollback 영역을 하나의 성공으로 간주

**제약**

- shell command 실행 자체는 구현하지 않는다.
- 이미 구조화된 evidence의 일관성 검증만 수행한다.
- 20~25분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 필수 evidence 하나씩 제거하면 모두 실패한다.
- [ ] SHA mismatch가 다른 성공으로 상쇄되지 않는다.
- [ ] backup checksum만 맞고 restore 대사가 다르면 실패한다.
- [ ] application/publication/database rollback 상태가 각각 독립적으로 검증된다.
- [ ] 실패 코드에 입력 원문이 반사되지 않는다.
- [ ] 승인 receipt가 bounded allowlist만 포함한다.
- [ ] evaluator가 external command를 직접 실행하지 않는다.

### 구현 후 설명할 것

- 세 rollback domain을 분리한 이유
  - **모범답변:** application rollback은 실행 artifact와 traffic, publication rollback은 current pointer, database recovery는 물리·논리 데이터 대상을 바꿉니다. 한 영역의 준비 상태가 다른 영역의 안전성을 증명하지 않으므로 evidence와 실행 권한을 독립적으로 확인해야 합니다.
- forward-only migration과 이전 application 호환성 전략
  - **모범답변:** 먼저 nullable column·새 table처럼 additive 변경을 배포하고 신·구 application이 함께 동작하는 기간을 둡니다. 이전 artifact가 최신 schema에서 smoke를 통과한 증거를 gate에 넣고, column 제거·constraint 강화는 구 code가 사라진 뒤 별도 release에서 수행합니다.
- checksum, manifest receipt, restore reconciliation의 역할 차이
  - **모범답변:** checksum은 읽은 bytes의 동일성, out-of-band manifest는 기대 digest가 backup과 함께 변조되지 않았다는 비교 기준, restore reconciliation은 그 bytes가 실제로 복원되어 row count·migration·current pointer·canonical fact set이 맞는지를 증명합니다.
- clean SHA와 immutable artifact가 incident response에 주는 이점
  - **모범답변:** exact clean SHA로 만든 immutable artifact는 어떤 code와 dependency를 실행했는지 재현하게 하고, 로컬 미커밋 변경이 섞인 모호성을 없앱니다. 장애 시 같은 artifact를 재배포하거나 이전 검증 artifact로 전환하는 판단도 빨라집니다.
- 자동 gate와 사람 checkpoint의 경계
  - **모범답변:** 형식·hash·설정·migration plan·restore 대사처럼 결정적인 조건은 자동화해 누락을 막습니다. 실제 배포 승인, production credential 접근, traffic 전환, destructive recovery 선택은 분리된 사람 권한과 runbook checkpoint에 남겨야 합니다.

### 원본 확인 위치

- Thread 01, Thread 19, Thread 22
- 커밋 `docs: define phase zero release gate`
- 커밋 `feat(ops): rehearse postgres recovery`
- 커밋 `fix(ops): harden postgres recovery boundaries`
- 커밋 `docs: record predeploy completion evidence`
- 파일 `docs/IMPLEMENTATION-PLAN.md`, `Makefile`, `scripts/secret_check.py`, `scripts/postgres_backup_restore.py`, `config/settings.py`
- 구성 요소 `production-check`, `validate_production_environment`, `create_backup`, `restore_backup`
