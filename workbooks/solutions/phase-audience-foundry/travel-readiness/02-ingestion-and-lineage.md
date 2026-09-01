# Ingestion 상태 기계와 증거 계보 워크북

이 문서는 외부 호출의 “사실”을 durable receipt로 남기고, 재시도·중단·권리 변경을 통과한 성공 입력만 content-addressed artifact와 ordered parse lineage로 승격하는 흐름을 다룬다.

<a id="w04"></a>

---

# [travel_windows/ingestion.py::DurableAviationAttemptRecorder] W04 — 외부 호출을 감싸는 durable 수집 영수증 (S)

## 면접 질문

`DurableAviationAttemptRecorder.__call__`은 `FetchAttempt(STARTED)`를 durable transaction으로 먼저 commit하고, 실제 HTTP 호출은 transaction 밖에서 실행한 뒤, 별도 durable transaction으로 결과를 닫습니다. 하나의 transaction 안에서 모두 처리하지 않는 이유와 이 구조가 보장하는 것·보장하지 못하는 것을 설명해 보세요.

꼬리 질문:

1. HTTP 성공 후 close transaction 전에 worker가 죽으면 어떤 상태가 남고, 후속 실행은 이를 어떻게 해석해야 하나요?
2. 성공 결과의 bytes/hash/length를 recorder가 다시 검증하는 이유는 무엇인가요?
3. 기존 `SourceArtifact`가 있으면 재사용하되 `first_successful_attempt`을 바꾸지 않는 이유는 무엇인가요?
4. `receipt_for`가 값 동등성보다 결과 객체 identity를 요구하는 이유는 무엇인가요?

## 30초 모범 답변

긴 네트워크 I/O 동안 DB transaction과 row lock을 잡지 않으면서도 “호출을 시작했다”는 사실은 먼저 durable하게 남기기 위한 구조입니다. 완료는 별도 transaction에서 STARTED를 정확히 한 번 종결하고, 성공이면 body hash 기준 artifact를 재사용하거나 생성합니다. 중간에 죽으면 STARTED가 남을 수 있으므로 다음 실행이 `WORKER_INTERRUPTED`로 reconcile합니다. 결과 객체 identity까지 묶어 나중에 만들어낸 같은 값이 실제 wire call의 receipt를 사칭하지 못하게 합니다.

## 답변 핵심 키워드

- durable intent receipt
- network outside transaction
- exactly-once close
- interruption reconciliation
- content addressing
- object provenance

## 꼬리 질문과 핵심 답변

### 왜 STARTED insert와 HTTP 호출 사이에도 작은 crash window가 허용되나요?

STARTED만 남는 것은 보수적으로 “시작 여부는 기록됐지만 완료를 증명하지 못함”으로 해석할 수 있습니다. 반대로 HTTP를 먼저 호출하면 외부 효과나 응답을 관찰했는데 DB에 아무 기록도 없는 더 위험한 구간이 생깁니다.

### DB transaction 안에서 HTTP를 호출하면 무엇이 나빠지나요?

느린 upstream 때문에 row lock과 connection이 오래 점유되고, DB timeout·deadlock·재시도와 외부 호출이 결합됩니다. transaction rollback이 이미 수행된 외부 I/O를 되돌리지도 못하므로 원자성 착시만 생깁니다.

### 성공 attempt와 artifact를 왜 분리하나요?

attempt는 언제·어떤 승인과 request fingerprint로 호출했는지 나타내는 사건이고, artifact는 source별 동일 content의 정체성입니다. 같은 body를 여러 번 받아도 attempt는 모두 남지만 artifact는 `(source, body hash)`로 하나일 수 있습니다.

### close 도중 persistence 오류가 나면 왜 원래 성공 body를 그대로 반환하면 안 되나요?

호출 결과와 durable receipt의 결합이 깨졌기 때문입니다. 상위 파서는 영속 계보가 확인된 결과만 사용해야 하므로 이 경우 전체 ingestion을 fail-closed해야 합니다.

### 객체 identity 검사가 프로세스 재시작을 견디나요?

아닙니다. 이것은 한 프로세스·한 수집 흐름 안에서 transport result와 receipt를 묶는 단기 capability입니다. 재시작 이후에는 DB의 attempt/artifact ID와 hash lineage가 영속 증거가 됩니다.

## 백지 구현

### 구현 목표

외부 함수 `call_once`를 durable STARTED/terminal receipt로 감싸는 executor를 구현한다. 외부 호출 중에는 DB transaction을 열어 두지 않고, 성공 body는 hash와 길이를 검증한 경우에만 content-addressed artifact와 연결한다.

### 인터페이스

`execute_recorded(source_code: str, request_id: SafeRequestIdentity, call_once: Callable[[], WireResult]) -> WireResult`

추가 메서드:

`receipt_for(source_code: str, result: WireResult) -> ReceiptRef`

repository는 `begin_durable()`, source/rights lock, attempt 생성·row lock·조건부 close, artifact 조회/생성을 제공한다고 가정한다.

```python
import hashlib


class PersistenceContractError(Exception):
    pass


def _valid_wire_result(result, request_id) -> bool:
    if type(result) is not WireResult or result.request_id != request_id:
        return False
    if result.outcome == "SUCCEEDED":
        return (
            type(result.body) is bytes
            and result.byte_count == len(result.body)
            and type(result.status) is int
            and 200 <= result.status <= 299
            and result.failure_code == ""
            and hashlib.sha256(result.body).hexdigest() == result.body_sha256
        )
    return (
        result.outcome in {"RETRYABLE_FAILED", "TERMINAL_FAILED"}
        and result.failure_code in ALLOWED_FAILURE_CODES
        and result.body == b""
        and result.byte_count == 0
        and result.body_sha256 == ""
    )


class DurableRecorder:
    def __init__(self, repository):
        self.repository = repository
        self._receipts: dict[int, ReceiptRef] = {}
        self._results: dict[int, WireResult] = {}
        self._source_codes: dict[int, str] = {}

    def _close_interrupted(self, attempt_id) -> None:
        try:
            with self.repository.begin_durable():
                self.repository.close_if_started(
                    attempt_id,
                    outcome="TERMINAL_FAILED",
                    failure_code="WORKER_INTERRUPTED",
                    status=None,
                    body_sha256="",
                    artifact_id=None,
                )
        except Exception:
            pass

    def execute_recorded(self, source_code, request_id, call_once):
        if source_code not in APPROVED_SOURCE_CODES or not callable(call_once):
            raise PersistenceContractError("SOURCE_GATE_FAILED")

        # 이 durable block은 callback이 시작되기 전에 끝나 commit된다.
        with self.repository.begin_durable():
            source, rights = self.repository.lock_approved_source(source_code)
            attempt_id = self.repository.create_started_attempt(
                source_id=source.id,
                source_revision=source.revision,
                rights_id=rights.id,
                request_id=request_id,
            )

        try:
            # 열린 transaction 없이 실제 I/O를 정확히 한 번 호출한다.
            result = call_once()
            if not _valid_wire_result(result, request_id):
                raise PersistenceContractError("PERSISTENCE_FAILED")

            with self.repository.begin_durable():
                attempt = self.repository.lock_attempt(attempt_id)
                if attempt.outcome != "STARTED":
                    raise PersistenceContractError("PERSISTENCE_FAILED")

                changed = self.repository.close_if_started(
                    attempt_id,
                    outcome=result.outcome,
                    failure_code=result.failure_code,
                    status=result.status,
                    body_sha256=result.body_sha256,
                    artifact_id=None,
                )
                if changed != 1:
                    raise PersistenceContractError("PERSISTENCE_FAILED")

                artifact_id = None
                if result.outcome == "SUCCEEDED":
                    artifact = self.repository.lock_artifact(
                        source.id, result.body_sha256
                    )
                    if artifact is not None:
                        if artifact.byte_count != result.byte_count:
                            raise PersistenceContractError("PERSISTENCE_FAILED")
                    else:
                        first_attempt = self.repository.first_successful_attempt(
                            source.id, result.body_sha256
                        )
                        if first_attempt is None:
                            raise PersistenceContractError("PERSISTENCE_FAILED")
                        artifact = self.repository.create_artifact(
                            source_id=source.id,
                            body_sha256=result.body_sha256,
                            byte_count=result.byte_count,
                            first_successful_attempt_id=first_attempt.id,
                        )
                    artifact_id = artifact.id
        except Exception:
            self._close_interrupted(attempt_id)
            raise

        key = id(result)
        self._results[key] = result
        self._source_codes[key] = source_code
        self._receipts[key] = ReceiptRef(attempt_id, artifact_id)
        return result

    def receipt_for(self, source_code, result):
        key = id(result)
        receipt = self._receipts.get(key)
        if (receipt is None or self._results.get(key) is not result
                or self._source_codes.get(key) != source_code):
            raise PersistenceContractError("PERSISTENCE_FAILED")
        return receipt
```

### 입력

- allowlist된 source code
- secret을 포함하지 않는 request identity
- 실제 외부 호출을 한 번 수행하는 callback

### 출력

- 검증된 `WireResult`
- 같은 프로세스 흐름에서만 조회 가능한 `ReceiptRef(attempt_id, artifact_id?)`
- 실패 attempt에는 artifact ID가 없다.

### 반드시 만족해야 할 조건

- callback 전에 STARTED attempt가 commit되어야 한다.
- callback 동안 DB transaction이 열려 있으면 안 된다.
- callback은 정확히 한 번만 실행되어야 한다.
- terminal close는 STARTED 행에만 성공하고 두 번째 close는 실패해야 한다.
- 성공 shape는 status, bytes type, byte count, SHA-256이 서로 일치해야 한다.
- 실패 shape는 body와 response hash가 비어 있어야 한다.
- callback 또는 close 예외 시 가능한 범위에서 STARTED를 `WORKER_INTERRUPTED`로 닫는다.
- 동일 source/body hash artifact가 있으면 재사용하되 길이 충돌은 거부한다.

### 경계 조건

- callback이 예외를 던짐
- callback이 잘못된 result 타입 또는 hash를 반환
- STARTED commit 뒤 callback 실행 전 중단
- callback 성공 뒤 terminal close 전 중단
- 동일 body의 동시 성공 attempt 두 개
- artifact는 존재하지만 byte count가 다름
- `receipt_for`에 값만 같은 새 result 객체를 전달

### 실패 조건

- external call 전에 receipt가 commit되지 않음
- network 동안 row lock/transaction을 유지함
- close 실패 후 body를 parse 단계로 넘김
- 실패 attempt에 artifact를 생성함
- raw body를 attempt/artifact 모델에 영속함

### 필요한 제약사항

- transaction은 durable한 짧은 단계로만 사용한다.
- conditional update 또는 row lock으로 STARTED → terminal 전이를 한 번만 허용한다.
- 반환·예외·로그에는 source allowlist code 외 임의 upstream 텍스트를 넣지 않는다.

## 구현 후 자가 검증

- [ ] STARTED가 callback보다 먼저 commit되는가?
- [ ] callback 시점에 열린 transaction이 없는가?
- [ ] 성공·retryable failure·terminal failure가 모두 정확히 한 번 종결되는가?
- [ ] callback/close 예외 후 STARTED가 가능한 한 interruption으로 닫히는가?
- [ ] 같은 content의 여러 attempt가 하나의 artifact를 공유하는가?
- [ ] artifact identity 충돌은 조용히 덮어쓰지 않고 실패하는가?
- [ ] receipt가 실제 반환 객체와 source code 모두에 결합되는가?
- [ ] DB에 raw body나 secret이 남지 않는가?

## 구현 후 설명할 것

- STARTED를 먼저 commit하는 선택이 crash ambiguity를 어느 방향으로 바꾸는지
  - 모범답변: STARTED를 먼저 commit하면 crash 뒤 “호출이 없었는지, 호출됐지만 응답 전에 죽었는지”는 구분하지 못해도 시도 예정 사실은 남습니다. 누락된 호출을 숨기는 대신 abandoned STARTED를 보수적으로 interruption으로 닫는 방향을 택한 것입니다.
- 외부 I/O와 DB transaction을 분리한 resource·concurrency 이유
  - 모범답변: network latency 동안 row lock과 connection을 점유하면 lock 대기와 pool 고갈이 커지고 장애가 DB로 전파됩니다. 시작과 종결만 짧은 durable transaction으로 만들면 장기 I/O와 데이터베이스 자원 lifetime을 분리할 수 있습니다.
- attempt와 artifact의 cardinality 및 책임 차이
  - 모범답변: attempt는 실제 호출마다 하나라 재시도와 실패까지 모두 기록합니다. artifact는 `(source, body hash)`별 하나라 동일 content를 여러 성공 attempt가 공유하며, byte count 충돌은 integrity 오류로 닫습니다.
- exactly-once 외부 호출이 아니라 exactly-once receipt close를 보장한다는 점
  - 모범답변: process crash와 원격 서버 사이에는 분산 transaction이 없어 외부 side effect의 exactly-once는 보장할 수 없습니다. 로컬에서는 STARTED 행의 조건부 전이로 terminal close가 한 번만 일어나도록 보장합니다.
- in-memory object identity와 durable hash lineage의 역할 차이
  - 모범답변: object identity는 현재 프로세스에서 callback이 반환한 바로 그 결과만 receipt와 결합해 결과 대체를 막습니다. durable hash는 프로세스를 넘어 같은 bytes의 provenance와 dedup을 증명하며 두 장치는 서로 대체할 수 없습니다.

## 원본 확인 위치

- 파일: `travel_windows/ingestion.py`
- 클래스/함수: `DurableAviationAttemptRecorder`, `_valid_attempt_result`, `_collected_input`, `_schedule_inputs`, `_reference_inputs`
- transport 결합: `sources/transport.py::_execute_aviation_attempt`
- Commit: `753690c feat(source): expose per-call aviation receipts`
- Commit: `95e4c46 feat(source): bind flight publication to collection receipts`
- 관련 테스트: `travel_windows/tests/test_source_publication.py::FlightEvidencePublicationTests`, 특히 failed real call receipt와 raw body 부재 테스트
- transport 테스트: `sources/tests/test_transport.py`, executor exact-once/결과 대체 방지/ordered lineage 테스트

<a id="w05"></a>

---

# [entry_requirements/ingestion.py::_run_entry_snapshot_ingestion] W05 — ingestion singleton·bounded retry·abandoned receipt 복구 (A)

## 면접 질문

입국 ingestion은 PostgreSQL session advisory lock을 autocommit connection에서 획득해 전체 수집 세션 동안 유지하고, 각 attempt만 별도 durable transaction으로 기록합니다. transaction-scoped lock 대신 session lock을 쓴 이유와 unlock 실패 시 connection을 폐기하는 이유를 설명해 보세요.

꼬리 질문:

1. 왜 실행 시작 직후 이전 `STARTED` attempt를 모두 `WORKER_INTERRUPTED`로 닫아야 하나요?
2. `max_retries=2`일 때 최대 실제 시도가 3회인 이유와 attempt 번호 연속성의 의미는 무엇인가요?
3. retryable 실패 allowlist가 필요한 이유는 무엇이며 인증 실패·schema 실패를 재시도하면 어떤 문제가 생기나요?
4. `KeyboardInterrupt`나 `SystemExit`도 receipt를 닫은 뒤 sanitized 형태로 다시 발생시키는 이유는 무엇인가요?

## 30초 모범 답변

수집 전체에는 네트워크와 여러 짧은 DB transaction이 있으므로 transaction lock으로 전체를 묶을 수 없습니다. session advisory lock으로 source당 worker 하나만 허용하고, autocommit을 강제해 실수로 긴 transaction에 들어가는 것을 막습니다. 이전 STARTED는 완료 증거가 없으므로 시작 시 interruption으로 reconcile합니다. 재시도는 timeout·429·5xx만 `max_retries + 1`회, 연속 번호로 남기며 lock 해제가 불확실하면 connection을 닫아 세션 lock 누수를 제거합니다.

## 답변 핵심 키워드

- session advisory lock
- autocommit precondition
- abandoned STARTED reconciliation
- bounded allowlist retry
- contiguous attempts
- connection discard

## 꼬리 질문과 핵심 답변

### session advisory lock이 process mutex보다 나은 점은 무엇인가요?

여러 프로세스·호스트가 같은 PostgreSQL을 사용해도 하나의 source 작업을 직렬화할 수 있습니다. process mutex는 한 worker 프로세스 안에서만 효과가 있습니다.

### lock 획득 실패와 DB 오류를 왜 같은 `ALREADY_RUNNING`으로 처리하면 안 되나요?

`pg_try_advisory_lock`이 정상적으로 false를 반환한 경우만 실제 overlap입니다. connection/SQL 오류를 overlap으로 숨기면 데이터베이스 장애를 정상적인 경쟁으로 오진하고 계속 운영할 수 있습니다.

### retry wait를 receipt transaction 안에서 실행하면 어떤 문제가 있나요?

sleep 동안 connection과 lock을 불필요하게 점유하고 다른 DB 작업을 막습니다. attempt close를 commit한 뒤 transaction 밖에서 bounded backoff해야 합니다.

### unlock 결과가 false이거나 예외인 경우 connection close가 왜 안전장치인가요?

session advisory lock의 생명주기는 connection입니다. 명시적 해제 여부를 증명할 수 없으면 connection을 끊는 것이 lock을 확실히 제거하는 유일한 fail-closed 방법입니다.

### 중단 예외를 일반 `Exception`으로 바꾸면 안 되는 이유는 무엇인가요?

호출자가 프로세스 종료·취소 의도를 알아야 합니다. 다만 원 traceback의 callback frame이나 secret이 노출될 수 있으므로 receipt를 terminal로 만든 뒤 chain 없는 새 process-control 예외를 발생시킵니다.

## 백지 구현

### 구현 목표

하나의 source ingestion만 동시에 실행되도록 session lock을 유지하면서, retryable 외부 실패를 제한적으로 재시도하고 모든 attempt를 terminal receipt로 남기는 작은 orchestration 상태 기계를 구현한다.

### 인터페이스

`run_ingestion(source_key: str, *, max_retries: int, fetch_once: Callable, parse_and_persist: Callable) -> IngestionOutcome`

DB adapter는 autocommit 여부, session lock try/unlock, connection close, STARTED 조회/종결, durable attempt create/close를 제공한다.

```python
import time


RETRYABLE = {"TIMEOUT", "RATE_LIMITED", "UPSTREAM_5XX"}


def _sanitized_process_control(error):
    raise type(error)() from None


def run_ingestion(source_key, *, max_retries, fetch_once, parse_and_persist):
    if type(max_retries) is not int or not 0 <= max_retries <= MAX_RETRIES:
        return IngestionOutcome("PERSISTENCE_FAILED", 0)
    if (db.vendor != "postgresql" or not db.autocommit
            or db.in_transaction):
        return IngestionOutcome("PERSISTENCE_FAILED", 0)

    acquired = False
    attempts = 0
    try:
        try:
            acquired = db.try_session_lock(source_key)
        except Exception:
            return IngestionOutcome("PERSISTENCE_FAILED", 0)
        if acquired is not True:
            return IngestionOutcome("ALREADY_RUNNING", 0)

        with db.begin_durable():
            db.close_all_started(
                source_key,
                outcome="TERMINAL_FAILED",
                failure_code="WORKER_INTERRUPTED",
            )

        for attempt_number in range(1, max_retries + 2):
            attempts = attempt_number
            with db.begin_durable():
                attempt_id = db.create_started_attempt(source_key, attempt_number)

            try:
                wire = fetch_once()
            except (KeyboardInterrupt, SystemExit, GeneratorExit) as error:
                try:
                    with db.begin_durable():
                        db.close_if_started(
                            attempt_id, "TERMINAL_FAILED", "WORKER_INTERRUPTED"
                        )
                except Exception:
                    pass
                _sanitized_process_control(error)
            except Exception:
                with db.begin_durable():
                    db.close_if_started(
                        attempt_id, "TERMINAL_FAILED", "WORKER_INTERRUPTED"
                    )
                return IngestionOutcome("PERSISTENCE_FAILED", attempts)

            # 다음 분기·sleep·parse 전에 terminal 상태가 commit된다.
            try:
                with db.begin_durable():
                    if db.close_attempt_from_wire(attempt_id, wire) != 1:
                        raise RuntimeError("closed attempt")
            except Exception:
                try:
                    with db.begin_durable():
                        db.close_if_started(
                            attempt_id, "TERMINAL_FAILED", "WORKER_INTERRUPTED"
                        )
                except Exception:
                    pass
                return IngestionOutcome("PERSISTENCE_FAILED", attempts)

            if wire.succeeded:
                try:
                    outcome = parse_and_persist(wire.body)
                except Exception:
                    return IngestionOutcome("PERSISTENCE_FAILED", attempts)
                if outcome not in {"REVIEW_REQUIRED", "REPLAY_OBSERVED"}:
                    return IngestionOutcome("PERSISTENCE_FAILED", attempts)
                return IngestionOutcome(outcome, attempts)

            if wire.failure_code not in RETRYABLE:
                return IngestionOutcome("FETCH_TERMINAL_FAILED", attempts)
            if attempt_number == max_retries + 1:
                return IngestionOutcome("FETCH_RETRIES_EXHAUSTED", attempts)

            # transaction 밖의 bounded exponential backoff
            time.sleep(min(BACKOFF_CAP_S, BACKOFF_BASE_S * (2 ** (attempt_number - 1))))

        raise AssertionError("bounded loop exhausted")
    except Exception:
        return IngestionOutcome("PERSISTENCE_FAILED", attempts)
    finally:
        if acquired:
            try:
                released = db.release_session_lock(source_key)
            except Exception:
                released = False
            if released is not True:
                db.discard_connection()
```

### 입력

- 고정 source key
- `0..N` 범위의 작은 max retries
- 한 번의 transport callback
- 성공 body를 parse/persist하는 callback

### 출력

- `REVIEW_REQUIRED`, `REPLAY_OBSERVED`, `FETCH_TERMINAL_FAILED`, `FETCH_RETRIES_EXHAUSTED`, `ALREADY_RUNNING`, `PERSISTENCE_FAILED` 같은 고정 outcome
- 실제 attempt count

### 반드시 만족해야 할 조건

- PostgreSQL·autocommit·not-in-transaction 조건을 lock 전에 확인한다.
- lock이 false면 receipt나 secret lookup, transport를 시작하지 않는다.
- lock 획득 후 같은 source의 과거 STARTED를 먼저 terminal interruption으로 닫는다.
- attempt 번호는 1부터 연속이고 최대 `max_retries + 1`이다.
- timeout, rate limit, upstream 5xx만 재시도한다.
- 각 attempt는 다음 sleep이나 parse 전에 terminal commit된다.
- 모든 반환/예외 경로에서 unlock을 시도하고, 성공을 확인할 수 없으면 connection을 폐기한다.

### 경계 조건

- `max_retries=0`
- 첫 시도 성공, 마지막 허용 시도 성공, 모두 retryable 실패
- terminal 인증 실패가 첫 시도에 발생
- lock 결과 false, lock SQL 오류, unlock false, unlock 예외
- 기존 STARTED가 여러 개 남아 있음
- transport에서 process-control 예외가 발생
- attempt close 자체가 실패한 뒤 재조정 필요

### 실패 조건

- overlap인데 새 receipt나 외부 호출을 시작함
- terminal 실패를 재시도함
- 최대 시도 수를 초과하거나 attempt 번호가 건너뜀
- retry sleep 중 transaction을 유지함
- unlock 불확실한 connection을 pool에 반환함

### 필요한 제약사항

- retry delay는 작은 지수 backoff와 고정 상한을 둔다.
- 외부 오류 문자열 대신 폐쇄형 code만 outcome에 포함한다.
- 원 body와 secret은 retry state나 예외에 저장하지 않는다.

## 구현 후 자가 검증

- [ ] 두 worker가 동시에 시작할 때 하나만 transport까지 도달하는가?
- [ ] DB 오류가 `ALREADY_RUNNING`으로 오분류되지 않는가?
- [ ] 과거 STARTED가 새 attempt보다 먼저 interruption으로 종결되는가?
- [ ] 최대 시도가 `max_retries + 1`이고 번호가 연속인가?
- [ ] retry allowlist 밖의 실패가 즉시 종료되는가?
- [ ] backoff에 상한이 있고 transaction 밖에서 실행되는가?
- [ ] unlock 실패 시 connection이 폐기되는가?
- [ ] process-control 예외는 terminal receipt 후 민감정보 없는 형태로 다시 발생하는가?

## 구현 후 설명할 것

- session lock과 transaction lock의 lifecycle 차이
  - 모범답변: transaction lock은 commit/rollback과 함께 풀리지만 PostgreSQL advisory session lock은 같은 connection에 붙어 전체 수집 동안 유지됩니다. 따라서 긴 network 구간을 직렬화하면서도 DB transaction은 짧게 끊을 수 있습니다.
- autocommit을 precondition으로 둔 이유
  - 모범답변: session lock SQL이 미완료 transaction 안에 숨어 있으면 이후 durable block과 unlock 시점이 모호해집니다. lock 전 autocommit·not-in-transaction을 확인해 lock lifetime을 connection lifetime과 정확히 맞춥니다.
- at-least-once 시도 기록과 bounded retry의 관계
  - 모범답변: 각 실제 호출 전에 STARTED를 남기므로 crash 복구까지 포함해 시도 기록은 at-least-once 성격입니다. 반면 현재 실행의 호출 수는 `max_retries + 1`로 제한하고 retryable code만 재시도해 폭주를 막습니다.
- abandoned STARTED를 실패로 닫는 보수적 복구 의미
  - 모범답변: 과거 worker가 원격 호출 전후 어느 지점에서 죽었는지 알 수 없으므로 성공으로 추정할 수 없습니다. 새 실행은 남은 STARTED를 `WORKER_INTERRUPTED`로 닫아 불확실성을 terminal failure로 명시합니다.
- unlock 불확실성을 connection lifecycle로 해결한 이유
  - 모범답변: unlock이 false이거나 예외면 lock이 connection에 남았는지 증명할 수 없습니다. 그 connection을 pool에 돌려보내지 않고 폐기하면 서버가 session 종료와 함께 lock을 확실히 회수합니다.

## 원본 확인 위치

- 대표 파일: `entry_requirements/ingestion.py`
- 함수: `_try_acquire_entry_ingestion_lock`, `_release_entry_ingestion_lock`, `_close_abandoned_started_attempts`, `_bounded_retry_wait`, `_run_entry_snapshot_ingestion`, `ingest_entry_snapshot`
- Commit: `fd78849 fix(entry): require durable ingestion sessions`
- 관련 테스트: `entry_requirements/tests/test_ingestion.py`, 특히 overlap/session lock/abandoned receipt/retry 테스트
- 동형 구현: `travel_warnings/ingestion.py::_try_acquire_warning_ingestion_lock`, `_close_abandoned_started_attempts`, `_run_travel_warning_ingestion`
- 후속 수정: `2d23c10 fix(warnings): reconcile prior ingestion attempts`
- 관련 테스트: `travel_warnings/tests/test_ingestion.py`, 특히 close/create interruption reconciliation 테스트

<a id="w06"></a>

---

# [sources/models.py::aviation_parse_input_identity] W06 — content-addressed artifact와 ordered multi-input parse lineage (S)

## 면접 질문

항공 schedule parser는 여러 departure/arrival 페이지를 하나의 `ParseRun`으로 묶습니다. 단순히 body hash 집합을 저장하지 않고 `(global ordinal, role, role sequence, artifact hash)`의 canonical identity와 `ParseRunInput` 행을 모두 남기는 이유를 설명해 보세요.

꼬리 질문:

1. global ordinal과 role별 sequence를 둘 다 요구하면 어떤 누락·순서 오류를 잡을 수 있나요?
2. 첫 input artifact를 `ParseRun.artifact`와 같게 강제하는 이유는 무엇인가요?
3. 같은 content를 다시 수집했을 때 ParseRun을 재사용하려면 어떤 필드를 비교해야 하나요?
4. Python에서 계산한 hash를 PostgreSQL trigger가 다시 계산하는 이유는 무엇인가요?

## 30초 모범 답변

여러 페이지는 집합이 아니라 역할과 순서를 가진 논리 입력입니다. body hash만 모으면 departure/arrival 교환, 중간 페이지 누락, 중복 페이지를 구분하기 어렵습니다. 전역 ordinal과 역할별 연속 sequence를 canonical serialization에 넣고 hash하며, 각 row도 보존해 감사와 재검증이 가능하게 합니다. ParseRun close 시 DB가 같은 identity를 재계산해 ORM 우회도 막고, replay는 parser contract·schema·ordered inputs가 정확히 같을 때만 idempotent하게 재사용합니다.

## 답변 핵심 키워드

- content-addressed artifact
- ordered logical input
- role sequence
- canonical hash
- idempotent replay
- DB recomputation

## 꼬리 질문과 핵심 답변

### hash만 저장하고 input row를 버리면 안 되나요?

hash는 동일성은 말해도 어떤 입력이 누락됐는지 설명하지 못합니다. input row는 provenance 조회, 역할별 연속성 검사, 첫 artifact 결합, 문제 발생 시 원인 격리에 필요합니다.

### artifact를 source별로 content-addressing하는 이유는 무엇인가요?

같은 bytes라도 서로 다른 공식 source에서 왔다면 권리·locator·의미가 다를 수 있습니다. 따라서 전역 hash 하나가 아니라 `(source, body hash)`가 identity여야 합니다.

### canonical serialization에서 delimiter와 domain prefix가 왜 중요한가요?

필드 경계를 모호하게 이어 붙이면 다른 tuple 목록이 같은 문자열이 될 수 있습니다. 명시적 delimiter·escaping 또는 길이 prefix가 필요하고, domain prefix는 같은 serialization을 다른 용도의 hash와 혼동하지 않게 합니다.

### ParseRunInput을 append-only로 만드는 이유는 무엇인가요?

parse 결과가 검토된 뒤 입력을 바꾸면 결과와 증거가 분리됩니다. STARTED 동안 연속 append만 허용하고 terminal close 뒤에는 어떤 추가·수정·삭제도 막아야 합니다.

### replay에서 결과만 같으면 재사용해도 되나요?

안 됩니다. 다른 input이나 parser version이 우연히 같은 typed 결과를 낼 수 있습니다. artifact/input identity, parser name/version/contract, expected·observed schema, terminal outcome까지 정확히 일치해야 동일 실행으로 볼 수 있습니다.

## 백지 구현

### 구현 목표

여러 content-addressed artifact를 역할과 순서가 있는 하나의 parse input으로 묶고, 누락·중복·순서 변경을 감지하는 canonical identity 계산기와 validator를 구현한다.

### 인터페이스

`input_identity(rows: Sequence[InputRef]) -> str`

`validate_parse_inputs(parser_kind: str, primary_artifact_id: str, rows: Sequence[InputRef], artifacts: Mapping[str, Artifact]) -> ValidationResult`

`InputRef`는 `ordinal`, `role`, `role_sequence`, `artifact_id`를 가진다. `Artifact`는 source ID, body SHA-256, state를 가진다.

```python
import hashlib
import re


IDENTITY_VERSION = "AVIATION_PARSE_INPUT_V1"
SHA256 = re.compile(r"^[0-9a-f]{64}$")
ROLE_RULES = {
    "schedule": frozenset({"SCHEDULE_DEPARTURE", "SCHEDULE_ARRIVAL"}),
    "city": frozenset({"DESTINATION_CITY"}),
    "legacy_arrivals": frozenset({"LEGACY_ARRIVAL"}),
}
ELIGIBLE_STATES = {"RECEIVED", "REVIEW_REQUIRED"}


def input_identity(rows):
    ordered = sorted(rows, key=lambda row: row.ordinal)
    lines = [IDENTITY_VERSION]
    next_role_sequence = {}
    for expected_ordinal, row in enumerate(ordered, 1):
        expected_sequence = next_role_sequence.get(row.role, 1)
        if (type(row.ordinal) is not int or row.ordinal != expected_ordinal
                or row.role not in set().union(*ROLE_RULES.values())
                or type(row.role_sequence) is not int
                or row.role_sequence != expected_sequence
                or SHA256.fullmatch(row.body_sha256 or "") is None):
            raise ValueError("invalid ordered parse input")
        lines.append(
            f"{row.ordinal}\t{row.role}\t{row.role_sequence}\t{row.body_sha256}"
        )
        next_role_sequence[row.role] = expected_sequence + 1
    if not ordered:
        raise ValueError("empty parse input")
    return hashlib.sha256(("\n".join(lines) + "\n").encode("utf-8")).hexdigest()


def validate_parse_inputs(parser_kind, primary_artifact_id, rows, artifacts):
    if not rows:
        return ValidationResult.failure("EMPTY")
    allowed = ROLE_RULES.get(parser_kind)
    if allowed is None:
        return ValidationResult.failure("ROLE_MISMATCH")

    ordered = sorted(rows, key=lambda row: row.ordinal)
    if [row.ordinal for row in ordered] != list(range(1, len(ordered) + 1)):
        return ValidationResult.failure("NON_CONTIGUOUS")
    if ordered[0].artifact_id != primary_artifact_id:
        return ValidationResult.failure("PRIMARY_MISMATCH")
    if any(row.role not in allowed for row in ordered):
        return ValidationResult.failure("ROLE_MISMATCH")
    roles = {row.role for row in ordered}
    if parser_kind == "schedule" and roles != allowed:
        return ValidationResult.failure("ROLE_MISMATCH")
    if parser_kind == "city" and len(ordered) != 1:
        return ValidationResult.failure("ROLE_MISMATCH")

    next_sequence = {}
    for row in ordered:
        expected = next_sequence.get(row.role, 1)
        if row.role_sequence != expected:
            return ValidationResult.failure("NON_CONTIGUOUS")
        next_sequence[row.role] = expected + 1

    primary = artifacts.get(primary_artifact_id)
    if primary is None or primary.state not in ELIGIBLE_STATES:
        return ValidationResult.failure("ARTIFACT_INELIGIBLE")
    normalized = []
    for row in ordered:
        artifact = artifacts.get(row.artifact_id)
        if artifact is None or artifact.state not in ELIGIBLE_STATES:
            return ValidationResult.failure("ARTIFACT_INELIGIBLE")
        if artifact.source_id != primary.source_id:
            return ValidationResult.failure("SOURCE_MISMATCH")
        if SHA256.fullmatch(artifact.body_sha256 or "") is None:
            return ValidationResult.failure("ARTIFACT_INELIGIBLE")
        normalized.append(InputRef(
            ordinal=row.ordinal,
            role=row.role,
            role_sequence=row.role_sequence,
            artifact_id=row.artifact_id,
            body_sha256=artifact.body_sha256,
        ))

    try:
        identity = input_identity(normalized)
    except ValueError:
        return ValidationResult.failure("NON_CONTIGUOUS")
    return ValidationResult.success(identity, len(normalized))
```

### 입력

- 적어도 한 개의 ordered input
- parser kind: schedule, city, legacy arrivals 중 하나
- primary artifact ID
- artifact metadata

### 출력

- 성공: canonical SHA-256와 정규화된 input 수
- 실패: `EMPTY`, `NON_CONTIGUOUS`, `ROLE_MISMATCH`, `SOURCE_MISMATCH`, `ARTIFACT_INELIGIBLE`, `PRIMARY_MISMATCH` 같은 고정 code

### 반드시 만족해야 할 조건

- ordinal은 1부터 전체 개수까지 중복 없이 연속이어야 한다.
- 각 role sequence도 1부터 해당 role 개수까지 연속이어야 한다.
- schedule parser는 departure와 arrival 역할을 모두 한 개 이상 포함해야 한다.
- city parser는 정확히 city input 하나만 허용한다.
- 모든 artifact는 primary와 같은 source이고 eligible state여야 한다.
- ordinal 1의 artifact가 primary artifact여야 한다.
- hash에는 ordinal, role, role sequence, body hash가 순서대로 모두 포함되어야 한다.

### 경계 조건

- 빈 입력
- ordinal 1이 없거나 중간 ordinal이 빠짐
- global 순서는 연속이지만 role sequence가 건너뜀
- 같은 artifact가 다른 역할에 반복됨
- departure만 있고 arrival이 없음
- body hash가 같지만 source가 다른 artifact
- row 입력 순서와 ordinal 정렬 순서가 다름

### 실패 조건

- 입력을 set으로 바꾸어 순서를 잃으면 안 된다.
- 문자열 단순 연결로 tuple 경계가 모호하면 안 된다.
- terminal ParseRun에 input을 추가할 수 있으면 안 된다.
- replay 시 hash 하나만 보고 parser contract 차이를 무시하면 안 된다.

### 필요한 제약사항

- canonical encoding은 명시적으로 문서화하고 deterministic해야 한다.
- 계산 시간은 input 수에 선형, 정렬이 필요하면 `O(n log n)` 이하여야 한다.
- raw body는 identity 계산에 사용하지 않고 검증된 body hash만 사용한다.

## 구현 후 자가 검증

- [ ] 같은 rows는 입력 container 순서와 무관하게 ordinal 기준 같은 hash를 내는가?
- [ ] role 교환, 중간 페이지 누락, 중복 sequence가 모두 다른 실패가 되는가?
- [ ] 첫 artifact와 primary가 다르면 거부되는가?
- [ ] 다른 source의 같은 body hash가 섞이면 거부되는가?
- [ ] schedule/city/legacy parser별 role allowlist가 적용되는가?
- [ ] terminal close 뒤 input 변경이 거부되는 상태 전이가 설계되어 있는가?
- [ ] Python과 DB에서 같은 canonical encoding을 재현할 수 있는가?

## 구현 후 설명할 것

- ordered input을 set/hash 목록으로 축약하지 않은 이유
  - 모범답변: 여러 page와 역할은 내용 hash가 같아도 위치 의미가 다릅니다. set으로 줄이면 누락·중복·role 교환·순서 변경을 구분할 수 없어 replay 증거가 약해집니다.
- global ordinal과 role sequence가 중복처럼 보여도 둘 다 필요한 이유
  - 모범답변: global ordinal은 전체 parse 입력의 canonical 결합 순서를 정하고, role sequence는 departure 1·2와 arrival 1·2처럼 역할 내부 page 연속성을 검증합니다. 하나만 있으면 다른 차원의 누락을 놓칩니다.
- content identity와 source provenance를 함께 확인하는 이유
  - 모범답변: SHA-256가 같다는 것은 bytes 동일성만 뜻하며 그 자료를 어느 승인 source에서 얻었는지는 말하지 않습니다. 같은 source revision·eligible artifact까지 확인해야 content와 provenance가 함께 닫힙니다.
- replay idempotency와 evidence conflict를 구분하는 기준
  - 모범답변: parser contract와 ordered input identity가 모두 같고 typed fingerprint도 같으면 replay로 봅니다. 같은 key에서 입력 identity나 typed 결과가 다르면 조용히 덮지 않고 evidence conflict로 격리합니다.
- Python·DB 이중 검증의 비용과 방어 범위
  - 모범답변: 애플리케이션 검증은 빠른 오류 분류와 읽기 쉬운 흐름을 주고, DB guard는 direct SQL·버그·경합 우회를 막습니다. 중복 비용은 있지만 terminal parse run 변경 같은 핵심 invariant를 저장 경계에서도 보장합니다.

## 원본 확인 위치

- 파일: `sources/models.py`
- 함수/클래스: `aviation_parse_input_identity`, `SourceArtifact`, `ParseRun`, `ParseRunInput`
- persistence: `travel_windows/ingestion.py::_persist_parse_run`, `_schedule_inputs`, `_reference_inputs`
- DB guard: `sources/migrations/0013_aviation_parse_inputs.py::sources_guard_parse_run_change`, `sources_guard_parse_run_input`
- Commit: `629bb0b feat(sources): bind ordered aviation parse inputs`
- 관련 테스트: `travel_windows/tests/test_source_publication.py`, ordered schedule inputs·schema drift·artifact state 테스트

<a id="w07"></a>

---

# [entry_requirements/ingestion.py::_locked_approved_source, _persist_success] W07 — 수집 전후의 exact source/rights 재검증 (A)

## 면접 질문

ingestion은 source와 최신 rights decision을 수집 전에 잠가 검증했는데도 성공 bytes를 typed revision으로 저장하기 직전에 다시 검증합니다. 첫 검증만으로 충분하지 않은 이유와 두 번째 검증 실패 시 어떤 데이터까지 남겨야 하는지 설명해 보세요.

꼬리 질문:

1. rights를 boolean `approved` 하나가 아니라 source revision, field scope, raw/typed storage, retention, attribution, contract fingerprint까지 exact match하는 이유는 무엇인가요?
2. 권리가 네트워크 중간에 revoke되면 이미 commit된 FetchAttempt와 artifact를 삭제해야 하나요?
3. latest decision을 sequence로 정하면서 append-only history를 유지하는 장점은 무엇인가요?
4. source registry의 check/apply command가 활성화 후 동일 계약을 다시 검증하는 이유는 무엇인가요?

## 30초 모범 답변

네트워크는 transaction 밖에서 실행되므로 첫 검증 이후 source 설정이나 rights가 바뀔 수 있습니다. 따라서 요청을 시작할 자격과 결과를 저장·게시할 자격을 별도로 확인해야 합니다. 두 번째 검증이 실패하면 수집 사건 receipt는 감사 증거로 남기되 raw body와 typed revision은 승격하지 않습니다. 승인 여부뿐 아니라 수집·저장·재게시·보존 범위와 정확한 contract hash를 비교해야 오래된 승인을 현재 스키마에 잘못 적용하지 않습니다.

## 답변 핵심 키워드

- TOCTOU
- exact contract gate
- append-only rights
- pre-fetch/post-fetch validation
- evidence vs typed storage
- fail-closed promotion

## 꼬리 질문과 핵심 답변

### 첫 transaction에서 rights row를 lock한 채 network를 호출하면 재검증이 없어도 되나요?

기술적으로 변경을 막을 수 있지만 긴 transaction과 lock 점유 문제가 생깁니다. 또한 운영자가 긴 upstream 지연 동안 revoke를 완료하지 못합니다. 짧은 첫 검증과 승격 직전 재검증이 더 나은 lifecycle 분리입니다.

### revoke 이후 기존 typed history도 지워야 하나요?

이 저장소의 정책은 append-only evidence/history이므로 자동 삭제하지 않습니다. 새 수집·승격을 막고, 과거 데이터의 표시·게시 여부는 명시적 retention/rollback 정책으로 처리해야 합니다. 무조건 삭제하면 감사 이력과 현재 포인터 정합성을 깨뜨릴 수 있습니다.

### `decision_sequence`만 높으면 최신이라고 믿어도 되나요?

source revision과 source ID 범위 안에서 sequence·ID 순서를 사용하고, DB 제약으로 append-only와 전이 규칙을 보장해야 합니다. sequence 중복이나 과거 revision 결정이 섞이면 exact gate에서 거부합니다.

### raw storage 권한이 false인데 artifact를 만드는 것은 모순 아닌가요?

artifact 행은 raw bytes가 아니라 source, body SHA-256, byte count, first attempt 같은 증거 metadata만 가집니다. raw body는 메모리에서 parse 후 폐기됩니다.

### contract fingerprint가 schema fingerprint와 다른 이유는 무엇인가요?

schema fingerprint는 데이터 구조를, contract fingerprint는 허용 source·field scope·parser/정책 버전 같은 더 넓은 사용 계약을 식별합니다. 구조가 같아도 권리 범위나 제공 endpoint가 달라질 수 있습니다.

## 백지 구현

### 구현 목표

긴 외부 I/O 전후에 source와 rights 계약을 검증하고, 중간 변경이 있으면 성공 bytes를 typed storage로 승격하지 않는 두 단계 gate를 구현한다.

### 인터페이스

`prepare_collection(source_code: str, expected: ContractSpec) -> CollectionLease`

`promote_result(lease: CollectionLease, result: VerifiedResult, expected: ContractSpec) -> PromotionOutcome`

`CollectionLease`에는 source ID/revision과 rights decision ID만 포함하며 secret이나 raw body는 포함하지 않는다.

```python
def _exact_contract(source, rights, expected) -> bool:
    return (
        source.code == expected.source_code
        and source.revision == expected.source_revision
        and source.state == "ACTIVE"
        and source.enabled is True
        and rights.decision == "APPROVED"
        and rights.access_allowed == expected.access_allowed
        and rights.automated_collection_allowed == expected.automated_collection_allowed
        and rights.typed_storage_allowed == expected.typed_storage_allowed
        and rights.raw_storage_allowed == expected.raw_storage_allowed
        and rights.typed_republication_allowed == expected.typed_republication_allowed
        and rights.raw_retention_seconds == expected.raw_retention_seconds
        and rights.typed_retention == expected.typed_retention
        and rights.evidence_retention == expected.evidence_retention
        and rights.field_scope_code == expected.field_scope_code
        and rights.attribution == expected.attribution
        and rights.contract_fingerprint_sha256 == expected.contract_hash
    )


def prepare_collection(source_code, expected):
    try:
        with repository.begin_durable():
            source = repository.lock_source_by_code(source_code)
            rights = repository.lock_latest_rights(source.id)
            if rights is None or not _exact_contract(source, rights, expected):
                return GateFailure("SOURCE_GATE_FAILED")
            return CollectionLease(
                source_id=source.id,
                source_revision=source.revision,
                rights_decision_id=rights.id,
            )
    except Exception:
        return GateFailure("SOURCE_GATE_FAILED")


def promote_result(lease, result, expected):
    # result에는 검증된 hash/typed projection만 있고 raw body 영속 필드는 없다.
    if type(result) is not VerifiedResult or not result.is_valid:
        return PromotionOutcome.failure("INVALID_RESULT")
    try:
        with repository.begin_durable():
            source = repository.lock_source_by_id(lease.source_id)
            latest = repository.lock_latest_rights(source.id)
            if (source.revision != lease.source_revision
                    or latest is None
                    or latest.id != lease.rights_decision_id
                    or not _exact_contract(source, latest, expected)):
                return PromotionOutcome.failure("RIGHTS_CHANGED")

            # 하나의 transaction에서 content-addressed artifact, parse run,
            # typed revision을 만들거나 동일 replay를 관찰한다.
            persisted = repository.persist_typed_projection(
                source_id=source.id,
                rights_id=latest.id,
                body_sha256=result.body_sha256,
                byte_count=result.byte_count,
                parse_identity=result.parse_identity,
                typed_fingerprint=result.typed_fingerprint,
                typed_facts=result.typed_facts,
            )
            return PromotionOutcome.success(
                artifact_id=persisted.artifact_id,
                parse_run_id=persisted.parse_run_id,
                typed_revision_id=persisted.typed_revision_id,
                replayed=persisted.replayed,
            )
    except ContractConflict:
        return PromotionOutcome.failure("RIGHTS_CHANGED")
    except Exception:
        return PromotionOutcome.failure("PERSISTENCE_FAILED")
```

### 입력

- 기대 source identity와 active/enabled 상태
- access, automated collection, typed storage, raw storage, republication, retention, scope, attribution, contract hash를 포함한 계약
- 검증된 transport result

### 출력

- 준비 성공: immutable lease
- 승격 성공: artifact/parse/typed revision 식별자
- 실패: `SOURCE_GATE_FAILED`, `RIGHTS_CHANGED`, `INVALID_RESULT`, `PERSISTENCE_FAILED`

### 반드시 만족해야 할 조건

- 준비 단계는 source와 해당 revision의 최신 rights를 lock해 exact match한다.
- 외부 호출 중 lock과 transaction을 유지하지 않는다.
- 승격 단계는 source·최신 rights를 다시 lock하고 동일 expected 계약인지 확인한다.
- lease의 rights가 더 이상 최신이 아니거나 source revision이 바뀌면 typed 저장을 하지 않는다.
- 실패하더라도 이미 commit된 attempt receipt는 삭제·수정하지 않는다.
- raw storage가 금지된 경우 bytes를 모델 필드나 실패 로그에 저장하지 않는다.

### 경계 조건

- fetch 중 source가 paused/disabled/revision upgrade됨
- 새 APPROVED decision이 추가됐지만 field scope가 다름
- REJECTED decision이 추가됨
- 계약 hash만 달라지고 나머지 flag는 같음
- 이전 rights ID는 같지만 source configuration 값이 drift함
- 같은 content가 이미 artifact로 존재함

### 실패 조건

- 첫 승인 snapshot만 믿고 두 번째 검증 없이 typed insert함
- `approved=True` 하나만 보고 retention/scope를 무시함
- gate 실패 시 과거 audit receipt를 삭제함
- raw bytes를 conflict 진단 목적으로 영속함

### 필요한 제약사항

- rights history는 append-only이며 latest 선택 규칙이 deterministic해야 한다.
- exact 비교 대상은 한 곳의 `ContractSpec`에서 관리한다.
- promotion transaction은 artifact·parse·typed projection을 원자적으로 처리한다.

## 구현 후 자가 검증

- [ ] 수집 전 정확한 source/rights만 lease를 받는가?
- [ ] network 중 transaction/row lock을 유지하지 않는가?
- [ ] source 또는 rights가 중간에 바뀌면 typed revision이 생성되지 않는가?
- [ ] attempt receipt는 gate 실패 후에도 terminal evidence로 남는가?
- [ ] raw storage 금지 조건이 DB schema·로그·예외까지 지켜지는가?
- [ ] 같은 내용 replay와 계약 conflict를 구분하는가?
- [ ] contract 비교 필드가 빠짐없이 중앙화되어 있는가?

## 구현 후 설명할 것

- TOCTOU를 긴 lock 대신 재검증으로 해결한 이유
  - 모범답변: network 동안 source와 rights row를 잠그면 운영자의 권리 철회가 오래 막히고 DB 자원이 고갈됩니다. 짧은 사전 snapshot을 lease로 잡고 승격 transaction에서 latest 상태를 다시 lock·exact compare해 중간 변경을 감지합니다.
- 수집 권리와 저장·재게시 권리를 분리한 이유
  - 모범답변: endpoint 접근 허용이 raw 보관이나 제품 재게시 허용을 자동으로 뜻하지 않습니다. automated collection, typed/raw storage, retention, republication을 각각 검사해야 최소 권한과 계약 범위를 지킬 수 있습니다.
- append-only rights history가 감사와 rollback에 주는 이점
  - 모범답변: 기존 결정을 수정하지 않고 새 결정을 추가하면 어떤 attempt가 당시 어느 계약에 근거했는지 재현할 수 있습니다. 최신 결정 선택도 결정적이고, 철회·업그레이드 이력이 사라지지 않습니다.
- gate 실패 시 receipt는 남기고 typed data는 막는 경계
  - 모범답변: 실제 호출이 있었다는 receipt는 감사 사실이므로 삭제하면 안 됩니다. 다만 수집 중 권리가 바뀌면 응답을 artifact·parse·typed projection으로 승격하지 않아 이후 게시 가능한 데이터가 생기지 않게 합니다.
- contract version upgrade를 조용한 in-place 변경으로 처리하지 않는 이유
  - 모범답변: field scope나 retention 의미가 바뀌면 같은 source code라도 다른 증거 계약입니다. revision과 fingerprint를 새 값으로 올려야 과거 데이터와 새 데이터가 섞이지 않고 replay·publication 검증도 정확해집니다.

## 원본 확인 위치

- 대표 파일: `entry_requirements/ingestion.py`
- 함수: `_latest_rights`, `_rights_are_exact`, `_source_is_exact`, `_locked_approved_source`, `_create_started_attempt`, `_persist_success`
- 동형 구현: `travel_warnings/ingestion.py::_locked_approved_source`, `_persist_success`, `_persist_snapshot_success`
- 모델: `sources/models.py::SourceConfiguration`, `SourceRightsDecision`
- registry: `sources/management/commands/register_approved_sources.py::_validate_rights_history`, `_apply_plan`, `register_approved_sources`
- Commit: `a3d779f feat(sources): add immutable source rights decisions`
- Commit: `3384af7 fix(warnings): bind candidates to current rights`
- 관련 테스트: `entry_requirements/tests/test_ingestion.py::test_rights_are_checked_before_fetch_and_again_before_persistence`
- 관련 테스트: `travel_warnings/tests/test_ingestion.py`의 midflight rights rejection·typed insert 직전 재검사 테스트
