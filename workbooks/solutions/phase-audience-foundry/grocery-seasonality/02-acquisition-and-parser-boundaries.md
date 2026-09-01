# 수집·요청·파서 경계

외부 I/O를 bounded evidence로 바꾸고 strict parser와 재현 가능한 parse run으로 연결한다.

> 이 문서는 정답 코드를 제공하지 않는다. 백지 구현은 원본을 다시 보기 전에 수행한다.

## P07. [Thread 03 / `feat(source): persist reconciled fetch receipts`] 원자적인 fetch 상태 전이와 영수증 대사

**우선순위:** S

### 면접 질문

- STARTED fetch를 SUCCEEDED로 바꿀 때 어떤 값들을 한 트랜잭션에서 대사해야 하나요?
- 페이지 영수증의 request ordinal, page number, declared total, 실제 row 수가 각각 왜 필요합니까?
- 성공 완료 명령을 재실행했을 때 언제 idempotent replay이고 언제 충돌인가요?
- 꼬리 질문: artifact 연결과 attempt 완료가 분리 커밋이면 어떤 중간 상태가 생길 수 있나요?
  - 모범답변: **프로젝트 특수사항:** `complete_kamis_fetch`는 영수증 생성, `SUCCEEDED` 전이와 집계 저장, hash-only artifact 생성·연결을 하나의 `transaction.atomic` 안에서 수행합니다. 이를 나누면 receipt만 생긴 `STARTED`, artifact가 없는 `SUCCEEDED`, 또는 artifact row는 생겼지만 attempt 연결이 빠진 상태가 남아 대사가 불가능해질 수 있습니다. **일반 원칙:** 상태 전이와 그 전이를 증명하는 증거의 연결은 하나의 원자적 consistency boundary로 묶어야 합니다.

### 30초 모범 답변

성공 전이는 attempt가 STARTED인지 잠근 뒤, 페이지 번호가 1부터 연속인지, provider total이 모든 페이지에서 일치하는지, 영수증 row 합과 결과 row 수가 같은지, ordered manifest와 budget이 맞는지를 대사합니다. 완료 상태 replay는 저장된 receipts·artifact·결과 hash가 모두 같을 때만 기존 결과를 반환하고, 하나라도 다르면 충돌로 실패합니다. attempt 완료와 artifact 연결은 같은 트랜잭션으로 묶어 성공인데 증거가 없는 상태를 막습니다.

### 답변 핵심 키워드

`state machine`, `transaction.atomic`, `select_for_update`, `receipt reconciliation`, `ordered manifest`, `idempotent replay`

### 백지 구현

**구현 목표**

메모리 모델로 축소한 fetch attempt를 STARTED에서 SUCCEEDED로 원자적으로 완료한다. 동일 replay는 허용하고 충돌 replay는 기존 상태를 보존한 채 거부한다.

**인터페이스 또는 함수 시그니처**

```python
def complete_fetch(
    attempt: FetchAttemptState,
    result: FetchResult,
) -> CompletedFetch:
    import hashlib
    import json
    import re
    from dataclasses import replace

    receipts = tuple(result.page_receipts)
    if not receipts:
        raise FetchValidationError("successful_fetch_requires_receipt")

    expected = tuple(range(1, len(receipts) + 1))
    if tuple(receipt.ordinal for receipt in receipts) != expected:
        raise FetchValidationError("receipt_ordinal_gap")
    if tuple(receipt.requested_page_number for receipt in receipts) != expected:
        raise FetchValidationError("page_number_gap")

    source = attempt.source_configuration
    if result.call_count < len(receipts):
        raise FetchValidationError("call_count_below_page_count")
    if result.call_count > source.max_requests_per_attempt:
        raise FetchValidationError("request_budget_exceeded")
    if len(receipts) > source.max_pages_per_attempt:
        raise FetchValidationError("page_budget_exceeded")

    sha256_pattern = re.compile(r"[0-9a-f]{64}\Z")
    for receipt in receipts:
        if (
            receipt.requested_page_number != receipt.declared_page_number
            or receipt.http_status != 200
            or receipt.provider_result_code != "0"
        ):
            raise FetchValidationError("unsuccessful_page_receipt")
        if receipt.byte_length < 0 or receipt.byte_length > source.max_page_bytes:
            raise FetchValidationError("page_byte_budget_exceeded")
        if sha256_pattern.fullmatch(receipt.body_sha256) is None:
            raise FetchValidationError("invalid_body_sha256")

    declared_totals = {receipt.declared_total_count for receipt in receipts}
    received_rows = sum(receipt.row_count for receipt in receipts)
    received_bytes = sum(receipt.byte_length for receipt in receipts)
    if len(declared_totals) != 1:
        raise FetchValidationError("declared_total_changed")
    if declared_totals != {len(result.rows)} or received_rows != len(result.rows):
        raise FetchValidationError("row_count_not_reconciled")

    # 원본과 같이 page 순서의 body hash 목록을 canonical manifest로 사용한다.
    # 길이는 receipt replay shape와 artifact.total_bytes에서 별도로 대사한다.
    manifest_payload = [receipt.body_sha256 for receipt in receipts]
    manifest_sha256 = hashlib.sha256(
        json.dumps(
            manifest_payload,
            ensure_ascii=True,
            sort_keys=True,
            separators=(",", ":"),
        ).encode("ascii")
    ).hexdigest()
    if manifest_sha256 != result.ordered_manifest_sha256:
        raise FetchValidationError("ordered_manifest_mismatch")

    artifact = SourceArtifact(
        ordered_manifest_sha256=manifest_sha256,
        page_count=len(receipts),
        total_bytes=received_bytes,
        retention_mode="HASH_ONLY",
    )
    completed_attempt = replace(
        attempt,
        state="SUCCEEDED",
        received_page_count=len(receipts),
        received_row_count=received_rows,
        received_byte_count=received_bytes,
        page_receipts=receipts,
        artifact=artifact,
    )
    candidate = CompletedFetch(
        attempt=completed_attempt,
        receipts=receipts,
        artifact=artifact,
    )

    if attempt.state == "SUCCEEDED":
        persisted = CompletedFetch(
            attempt=attempt,
            receipts=tuple(attempt.page_receipts),
            artifact=attempt.artifact,
        )
        persisted_shape = tuple(
            (
                receipt.ordinal,
                receipt.requested_page_number,
                receipt.declared_page_number,
                receipt.declared_total_count,
                receipt.row_count,
                receipt.byte_length,
                receipt.body_sha256,
            )
            for receipt in persisted.receipts
        )
        candidate_shape = tuple(
            (
                receipt.ordinal,
                receipt.requested_page_number,
                receipt.declared_page_number,
                receipt.declared_total_count,
                receipt.row_count,
                receipt.byte_length,
                receipt.body_sha256,
            )
            for receipt in receipts
        )
        if (
            persisted_shape != candidate_shape
            or persisted.artifact is None
            or persisted.artifact.ordered_manifest_sha256 != manifest_sha256
            or attempt.received_page_count != len(receipts)
            or attempt.received_row_count != received_rows
            or attempt.received_byte_count != received_bytes
        ):
            raise FetchConflict("completed_fetch_replay_conflict")
        return persisted

    if attempt.state != "STARTED":
        raise FetchConflict("fetch_already_finalized")

    # 입력 객체는 건드리지 않았으므로 여기까지의 어떤 실패도 부분 상태를 남기지 않는다.
    return candidate
```

**입력과 출력**

- 입력: STARTED attempt와 page receipts/rows/manifest를 가진 결과
- 출력: SUCCEEDED attempt, 검증된 receipts, hash-only artifact를 묶은 `CompletedFetch`

**반드시 만족해야 할 조건**

- 요청 ordinal과 page number가 1부터 빈틈없이 이어져야 한다.
- 모든 페이지의 declared total이 동일하고 전체 row 수와 일치해야 한다.
- 페이지 row·byte 합이 attempt 집계와 일치해야 한다.
- ordered manifest는 page 순서와 body hash/길이를 canonical하게 반영해야 한다.
- 첫 완료는 한 번만 상태를 전이한다.
- 완료 replay는 모든 의미 필드가 동일할 때만 기존 결과를 반환한다.

**경계 조건**

- 한 페이지 결과
- 0 row 성공 응답을 계약상 허용하거나 거부하는 경계
- 영수증 입력 순서가 뒤섞인 경우
- 완료 직후 동일 replay

**실패 조건**

- page gap 또는 중복
- provider total drift
- row/byte/hash 불일치
- FAILED 또는 SUCCEEDED attempt에 다른 결과로 완료 시도
- budget 초과

**제약**

- 원문 body를 저장하지 않고 hash/길이/메타데이터만 저장한다.
- 검증 실패 시 상태·receipt·artifact 중 일부만 변경되어서는 안 된다.
- 25~30분 이내 구현한다.

### 구현 후 자가 검증

- [ ] 정상 다중 페이지가 정확히 한 번 완료된다.
- [ ] 동일 replay가 새 artifact나 receipt를 만들지 않는다.
- [ ] 충돌 replay 후 기존 완료 결과가 변하지 않는다.
- [ ] 페이지 gap과 total drift가 검출된다.
- [ ] manifest가 page 순서 변화에 민감하다.
- [ ] 실패 시 부분 상태 변화가 없다.

### 구현 후 설명할 것

- 상태 머신과 DB constraint의 역할 분담
  - 모범답변: **프로젝트에서는** 서비스가 잠근 `STARTED` attempt에 대해서만 영수증 전체와 artifact를 교차 대사한 뒤 상태를 전이합니다. DB constraint는 상태별 완료 시각·실패 필드 형태, 영수증 ordinal/page 유일성, artifact의 source/manifest 유일성처럼 한 행 또는 고유 키 수준의 불변식을 마지막 방어선으로 보장합니다. 즉 서비스는 여러 행에 걸친 의미 검증을, DB는 우회 쓰기와 경합에도 깨지면 안 되는 구조적 불변식을 맡습니다.
- idempotent replay를 단순 key dedupe보다 엄격하게 정의한 이유
  - 모범답변: 같은 attempt 식별자는 같은 명령을 가리킬 뿐 같은 결과임을 증명하지 않습니다. 원본은 저장된 receipt의 ordinal·page·total·row·byte·body hash와 artifact manifest까지 후보 결과와 비교합니다. key만 같다고 성공시키면 재시도 과정의 다른 응답이나 비결정적 결과가 기존 성공으로 위장될 수 있습니다.
- hash-only retention의 감사 가능성과 원문 미보존 trade-off
  - 모범답변: 페이지별 hash·길이·상태와 ordered manifest를 남기면 동일 바이트의 재관측, 페이지 순서, count 대사와 content-addressed dedupe는 검증할 수 있습니다. 반면 원문을 보존하지 않으므로 나중에 새 parser로 재처리하거나 특정 필드 drift를 직접 조사할 수 없고 다시 수집해야 합니다. 프로젝트는 이 재현 비용을 감수하고 저장·노출 위험을 낮추는 `HASH_ONLY`를 고정했습니다.
- select-for-update가 보호하는 race와 unique constraint가 보호하는 race
  - 모범답변: `select_for_update`는 같은 attempt를 동시에 완료·실패시키는 트랜잭션을 직렬화해 둘 다 `STARTED`를 보고 전이하는 race를 막습니다. unique constraint는 `(acquisition_run_id, attempt_ordinal)`, attempt별 receipt ordinal/page, `(source_identity, ordered_manifest_sha256)` 중복을 DB 수준에서 막아 다른 코드 경로나 동시 insert에도 중복 증거가 생기지 않게 합니다.

### 원본 확인 위치

- Thread 03
- 커밋 `feat(source): persist reconciled fetch receipts`
- 파일 `grocery/source/persistence.py`
- 구성 요소 `CompletedKamisFetch`, `start_kamis_fetch`, `complete_kamis_fetch`, `fail_kamis_fetch`
- 연관 Thread 04, 09, 10

## P08. [Thread 03 / `feat(source): retain partial failure receipts`] 부분 네트워크 실패의 증거 보존과 실패 폐쇄

**우선순위:** S

### 면접 질문

- 3페이지 중 2페이지까지 받은 뒤 timeout이 나면 무엇을 남기고 무엇을 남기지 않아야 하나요?
- 예외 객체에 partial receipts를 담는 설계의 장점과 위험은 무엇인가요?
- 실패 코드에 provider 응답·URL·credential 일부를 그대로 넣으면 왜 안 되나요?
- 꼬리 질문: 실패 처리 자체가 budget 검증에서 실패하면 최종 상태를 어떻게 보장하나요?
  - 모범답변: **프로젝트 특수사항:** `fail_kamis_fetch`도 원자 트랜잭션 안에서 partial receipt를 다시 검증한 뒤에만 실패 상태를 저장합니다. budget이나 receipt shape 검증이 실패하면 전체가 롤백되어 attempt는 `STARTED`, receipt는 0개로 유지되므로 검증되지 않은 증거로 억지로 `FAILED`를 만들지 않습니다. **일반 원칙:** 실패 기록 경로 자체가 실패하면 이전의 일관된 상태를 보존하고, 고정된 안전 코드와 검증 가능한 증거로 재시도하거나 운영 경보를 올려야 합니다.

### 30초 모범 답변

부분 실패도 감사 대상이므로 이미 완료된 페이지의 순서·상태·길이·hash는 보존하되 원문과 민감한 요청 값은 남기지 않습니다. transport 예외는 고정된 안전 코드와 검증 가능한 partial receipt만 전달하고, persistence 계층이 attempt를 잠근 뒤 receipt budget과 연속성을 다시 검증해 FAILED로 한 번만 전이합니다. 실패 저장 중 검증이 깨지면 전체 트랜잭션을 롤백해 반쯤 실패한 상태를 만들지 않습니다.

### 답변 핵심 키워드

`partial receipts`, `safe error code`, `redaction`, `atomic failure finalization`, `no double finalize`, `rollback`

### 백지 구현

**구현 목표**

부분 영수증을 가진 transport 실패를 받아 attempt를 FAILED로 원자적으로 마감하는 함수를 구현한다.

**인터페이스 또는 함수 시그니처**

```python
def fail_fetch(
    attempt: FetchAttemptState,
    error: TransportFailure,
) -> FailedFetch:
    import re
    from dataclasses import replace

    # str(error)는 URL이나 provider 문구를 포함할 수 있으므로 절대 저장하지 않는다.
    if type(error.code) is not str or error.code not in _KNOWN_TRANSPORT_CODES:
        raise FetchValidationError("unsafe_failure_code")
    receipts = error.partial_page_receipts
    if not isinstance(receipts, tuple) or any(
        not isinstance(receipt, PageReceipt) for receipt in receipts
    ):
        raise FetchValidationError("invalid_partial_receipt_container")

    source = attempt.source_configuration
    if error.page_number is not None:
        if type(error.page_number) is not int or error.page_number < 1:
            raise FetchValidationError("failed_page_invalid")
        if error.page_number > source.max_pages_per_attempt:
            raise FetchValidationError("failed_page_budget_exceeded")
    if error.attempt is not None:
        if type(error.attempt) is not int or error.attempt < 1:
            raise FetchValidationError("failed_attempt_invalid")
        if (
            error.attempt > source.max_retries + 1
            or error.attempt > source.max_requests_per_attempt
        ):
            raise FetchValidationError("failed_request_budget_exceeded")

    if receipts:
        expected = tuple(range(1, len(receipts) + 1))
        if tuple(receipt.ordinal for receipt in receipts) != expected:
            raise FetchValidationError("partial_receipt_ordinal_gap")
        if tuple(receipt.requested_page_number for receipt in receipts) != expected:
            raise FetchValidationError("partial_receipt_page_gap")
        if error.page_number != len(receipts) + 1:
            raise FetchValidationError("partial_receipts_not_failure_prefix")
        if len(receipts) > min(
            source.max_pages_per_attempt,
            source.max_requests_per_attempt,
        ):
            raise FetchValidationError("partial_receipt_budget_exceeded")
        if len(receipts) + (error.attempt or 1) > source.max_requests_per_attempt:
            raise FetchValidationError("partial_request_budget_exceeded")

        sha256_pattern = re.compile(r"[0-9a-f]{64}\Z")
        for receipt in receipts:
            # 실패한 page 자체는 넣지 않고, body를 완전히 받은 성공 page만 보존한다.
            integer_values = (
                receipt.ordinal,
                receipt.requested_page_number,
                receipt.declared_page_number,
                receipt.declared_page_size,
                receipt.declared_total_count,
                receipt.row_count,
                receipt.http_status,
                receipt.byte_length,
            )
            if (
                any(
                    not isinstance(value, int) or isinstance(value, bool)
                    for value in integer_values
                )
                or receipt.requested_page_number != receipt.declared_page_number
                or receipt.declared_page_size <= 0
                or receipt.declared_total_count <= 0
                or receipt.row_count != receipt.declared_page_size
                or receipt.http_status != 200
                or receipt.provider_result_code != "0"
                or receipt.byte_length <= 0
                or receipt.byte_length > source.max_page_bytes
                or not isinstance(receipt.body_sha256, str)
                or sha256_pattern.fullmatch(receipt.body_sha256) is None
            ):
                raise FetchValidationError("invalid_partial_receipt_shape")

        totals = {receipt.declared_total_count for receipt in receipts}
        sizes = {receipt.declared_page_size for receipt in receipts}
        if len(totals) != 1 or len(sizes) != 1:
            raise FetchValidationError("partial_pagination_drift")
        declared_total = next(iter(totals))
        declared_page_size = next(iter(sizes))
        required_pages = max(
            1,
            (declared_total + declared_page_size - 1) // declared_page_size,
        )
        if required_pages > min(
            source.max_pages_per_attempt,
            source.max_requests_per_attempt,
        ):
            raise FetchValidationError("declared_total_exceeds_budget")
        if sum(receipt.row_count for receipt in receipts) >= declared_total:
            raise FetchValidationError("partial_rows_not_incomplete")

    row_count = sum(receipt.row_count for receipt in receipts)
    byte_count = sum(receipt.byte_length for receipt in receipts)
    candidate_attempt = replace(
        attempt,
        state="FAILED",
        failure_code=error.code.upper(),
        received_page_count=len(receipts),
        received_row_count=row_count,
        received_byte_count=byte_count,
        page_receipts=receipts,
    )
    candidate = FailedFetch(attempt=candidate_attempt, receipts=receipts)

    if attempt.state == "FAILED":
        persisted_receipts = tuple(attempt.page_receipts)
        if (
            attempt.failure_code != candidate_attempt.failure_code
            or persisted_receipts != receipts
            or attempt.received_page_count != len(receipts)
            or attempt.received_row_count != row_count
            or attempt.received_byte_count != byte_count
        ):
            raise FetchConflict("failed_fetch_replay_conflict")
        return FailedFetch(attempt=attempt, receipts=persisted_receipts)
    if attempt.state != "STARTED":
        raise FetchConflict("fetch_already_finalized")

    # 모든 검증 뒤에 새 값을 만들기 때문에 실패 중간 상태가 입력에 기록되지 않는다.
    return candidate
```

**입력과 출력**

- 입력: STARTED attempt, 안전한 failure code, 완료된 페이지들의 partial receipt
- 출력: FAILED attempt와 저장된 partial receipts

**반드시 만족해야 할 조건**

- 부분 receipt는 ordinal이 1부터 연속이고 각 body 상태가 자기 필드와 일치해야 한다.
- 실패 코드만 저장하며 원본 예외 메시지·URL·query·credential을 저장하지 않는다.
- row/byte/page budget을 초과하면 어떤 상태도 변경하지 않는다.
- STARTED attempt만 최초 실패 전이를 허용한다.
- 동일 실패 replay는 저장된 증거가 같을 때만 허용한다.
- SUCCEEDED attempt를 실패로 덮어쓰지 않는다.

**경계 조건**

- 첫 요청 전 timeout으로 receipt가 하나도 없는 경우
- 한 페이지 body를 받기 전 연결 실패
- 마지막 페이지 직후 persistence 전에 실패한 경우
- 동일 실패 명령 재시도

**실패 조건**

- partial receipt gap 또는 중복
- body 미수신 상태인데 body hash/bytes가 있는 모순
- 안전 코드 allowlist 밖의 문자열
- 이미 다른 증거로 finalized 된 attempt

**제약**

- 원본 예외의 `str()`을 외부 출력이나 저장 필드로 사용하지 않는다.
- 성공 완료 함수와 동일한 receipt validator를 재사용할 수 있게 경계를 설계한다.
- 20~25분 구현 크기다.

### 구현 후 자가 검증

- [ ] 0개·1개·여러 개 partial receipt가 각각 처리된다.
- [ ] 실패 코드 외의 비밀 marker가 결과에 포함되지 않는다.
- [ ] 모순된 receipt가 attempt를 FAILED로 바꾸지 않는다.
- [ ] 동일 replay와 충돌 replay가 구분된다.
- [ ] 성공 attempt는 실패 처리로 변경되지 않는다.
- [ ] 예외 경로에서도 mutable 입력을 변경하지 않는다.

### 구현 후 설명할 것

- 부분 실패 증거가 운영 디버깅과 재시도 판단에 주는 가치
  - 모범답변: 완료된 page의 ordinal·row 수·byte 길이·body hash를 남기면 어느 prefix까지 정상 수신했고 어느 page에서 실패했는지, quota와 budget을 얼마나 썼는지 확인할 수 있습니다. 원본은 실패한 page의 불완전 body는 저장하지 않고 완전히 검증된 page receipt만 남겨, 다음 시도를 판단할 근거와 증거의 신뢰도를 함께 지킵니다.
- 예외를 data carrier로 쓸 때의 검증 경계
  - 모범답변: `KamisTransportError`는 안전 코드와 제한된 정수 상태, raw-free receipt tuple을 transport에서 persistence로 전달합니다. 하지만 예외 객체도 호출자에게 노출된 입력이므로 persistence가 타입, 연속성, total/page-size 일관성, hash와 budget을 다시 검증합니다. 예외에 담겼다는 이유만으로 신뢰하면 변조되거나 모순된 증거가 영속화될 수 있습니다.
- 실패 코드 정규화가 보안 경계인 이유
  - 모범답변: 원본 예외 메시지에는 URL, query, credential, provider 문구가 섞일 수 있습니다. 프로젝트는 allowlist의 고정 코드를 대문자로 저장하고 알 수 없는 값은 안전한 분류로 제한하므로, DB·로그·API 응답으로 비밀이 전파되는 경로를 끊습니다. 분류에 필요한 HTTP/provider 값도 범위와 형식을 제한합니다.
- 실패 persistence까지 원자적이어야 하는 이유
  - 모범답변: partial receipt만 저장되고 attempt가 `STARTED`로 남거나, attempt count와 receipt 합이 다른 실패 상태가 되면 재시도와 감사 판단이 모두 모호해집니다. 원본은 receipt insert, 실패 상태·분류·집계·완료 시각 저장을 한 트랜잭션으로 묶어 검증 오류나 DB 오류 시 전부 롤백합니다.

### 원본 확인 위치

- Thread 03
- 커밋 `feat(source): retain partial failure receipts`
- 파일 `grocery/source/client.py`
- 파일 `grocery/source/persistence.py`
- 구성 요소 `KamisTransportError`, `partial_page_receipts`, `fail_kamis_fetch`

## P09. [Thread 03 / `feat(source): persist historical request scopes`] 요청 allowlist·redaction·semantic scope hash

**우선순위:** A

### 면접 질문

- HTTP 요청 전체 URL을 감사 로그나 DB에 저장하지 않고도 동일 요청 범위를 식별하려면 어떻게 해야 하나요?
- scope hash에 secret을 넣지 않으면서도 dataset·mode·기간·region 변경에 민감하게 만드는 방법은 무엇인가요?
- 승인된 endpoint 계약을 애플리케이션 검증과 DB constraint에 함께 고정한 이유는 무엇인가요?
- 꼬리 질문: `repr()`이나 예외 메시지에서 query parameter가 새는 것을 어떻게 테스트하겠습니까?
  - 모범답변: **프로젝트 특수사항:** `HistoricalPriceQuery`와 `PreparedHistoricalRequest.query`는 repr에서 제외되고, `ValidatedHistoricalQuery.__repr__`도 dataset과 condition 이름만 출력합니다. 테스트에서는 기간·region·credential에 고유한 secret marker를 넣은 뒤 정상 객체의 `repr`, 모든 검증 예외, request shape에 marker가 없는지 확인하고, 원본처럼 오류가 고정 코드만 갖는지도 검사합니다. **일반 원칙:** 민감 값 비노출은 예시 한두 개가 아니라 성공·실패·로깅·직렬화 경로를 아우르는 negative security test로 고정해야 합니다.

### 30초 모범 답변

요청은 승인된 dataset·mode·host·path와 bounded query field만 허용하고 credential은 별도 전달합니다. 감사용 scope hash는 secret을 제외한 semantic 요청 필드와 계약 버전을 canonical하게 직렬화해 계산하므로 같은 범위는 안정적으로 식별하면서 기간·region·dataset 변경은 감지합니다. 객체 표현과 오류는 고정 코드와 안전 필드만 노출하고, endpoint 계약은 DB에도 고정해 우회 저장을 막습니다.

### 답변 핵심 키워드

`allowlist`, `semantic scope hash`, `credential separation`, `redacted repr`, `endpoint pinning`, `canonical request`

### 백지 구현

**구현 목표**

승인된 역사 데이터 요청을 검증하고, 비밀을 포함하지 않는 canonical scope hash와 redacted 표현을 만든다.

**인터페이스 또는 함수 시그니처**

```python
def prepare_historical_request(
    contract: EndpointContract,
    query: HistoricalQuery,
) -> PreparedHistoricalRequest:
    import hashlib
    import json

    modes = {
        "15156060": "HISTORICAL_MONTHLY",
        "15156062": "HISTORICAL_REGIONAL",
        "15156065": "HISTORICAL_MARKET",
    }
    paths = {
        "15156060": "/B552845/perYearMonth/price",
        "15156062": "/B552845/perRegion/price",
        "15156065": "/B552845/periodRetail/price",
    }
    dataset_id = contract.dataset_id
    if dataset_id not in modes:
        raise HistoricalContractError("invalid_historical_dataset")
    if (
        contract.publication_mode != modes[dataset_id]
        or contract.endpoint_scheme != "https"
        or contract.endpoint_host != "apis.data.go.kr"
        or contract.endpoint_path != paths[dataset_id]
        or contract.endpoint_method != "GET"
        or contract.authentication_mode != "DATA_GO_KR_SERVICE_KEY"
    ):
        raise HistoricalContractError("unapproved_historical_endpoint")

    # 원본 validate_historical_query가 월/일 형식·기간 상한·필터 계층과 코드를 검증한다.
    dataset = HistoricalDataset(dataset_id)
    validated = validate_historical_query(dataset, query)
    endpoint_contract = HISTORICAL_ENDPOINT_CONTRACTS[dataset]
    condition_names = frozenset(validated.conditions)
    if not condition_names <= endpoint_contract.allowed_condition_names:
        raise HistoricalContractError("request_parameter_allowlist_violation")

    common_names = frozenset({"numOfRows", "pageNo", "returnType"})
    names = sorted(common_names | condition_names)
    names.append("serviceKey:<redacted>")
    request_shape = f"GET {contract.endpoint_path} parameters=[{','.join(names)}]"

    # scope format과 source revision을 포함해 canonical 계약 변경도 별도 scope가 되게 한다.
    canonical_scope = {
        "conditions": sorted(validated.conditions.items()),
        "configuration_revision": contract.configuration_revision,
        "dataset": dataset_id,
        "publication_mode": contract.publication_mode,
        "scope_format": 1,
    }
    scope_sha256 = hashlib.sha256(
        json.dumps(
            canonical_scope,
            ensure_ascii=True,
            sort_keys=True,
            separators=(",", ":"),
        ).encode("ascii")
    ).hexdigest()

    # service key는 build 호출 시에만 주입되므로 준비 객체·hash·repr 어디에도 없다.
    return PreparedHistoricalRequest(
        query=validated,
        request_shape=request_shape,
        scope_sha256=scope_sha256,
    )
```

**입력과 출력**

- 입력: 승인된 endpoint contract와 기간/region 등의 typed query
- 출력: 전송용 안전 필드와 64자리 `scope_sha256`을 가진 준비 객체

**반드시 만족해야 할 조건**

- dataset와 publication mode, host/path, auth mode의 승인 조합만 허용한다.
- 기간·페이지·행 수 등 각 query field를 canonical 형식과 상한으로 검증한다.
- secret 값은 scope hash, repr, equality debug output에 포함하지 않는다.
- scope hash는 semantic field와 contract version 변화에 민감하다.
- mapping key 순서에 의존하지 않는다.

**경계 조건**

- 동일 query를 다른 dict 순서로 구성한 경우
- 날짜 범위의 양 끝 경계
- 선행 0이 있는 지역 코드
- 월 단위 query와 일 단위 query의 구분

**실패 조건**

- 승인되지 않은 host/path/dataset 조합
- unknown query parameter 또는 duplicate parameter
- 상한을 넘는 기간/페이지 크기
- repr 또는 예외에 secret marker가 포함됨

**제약**

- 실제 HTTP 호출은 구현하지 않는다.
- secret을 dummy 값으로 scope에 넣는 방식도 금지한다.
- 20분 이내 순수 검증/준비 함수로 작성한다.

### 구현 후 자가 검증

- [ ] 동일 semantic query의 scope hash가 안정적이다.
- [ ] 기간·region·dataset 하나가 바뀌면 hash가 바뀐다.
- [ ] secret 값이 어디에도 직렬화되지 않는다.
- [ ] unknown field가 조용히 무시되지 않는다.
- [ ] 월/일 query 계약이 서로 섞이지 않는다.

### 구현 후 설명할 것

- full URL 저장과 semantic receipt 저장의 보안·감사 trade-off
  - 모범답변: full URL은 요청을 그대로 재현하기 쉽지만 service key와 기간·region 같은 조건 값이 DB·로그·백업에 장기 잔존할 수 있습니다. 프로젝트는 값 없는 request shape와 secret을 제외한 semantic scope hash를 저장해 같은 범위인지 감사할 수 있게 했습니다. 대신 DB만 보고 원래 query 값을 복원하거나 세부 요청을 디버깅할 수는 없다는 비용이 있습니다.
- application allowlist와 DB constraint를 함께 둔 이유
  - 모범답변: 애플리케이션의 `validate_historical_query`는 날짜 범위, 필터 계층, dataset별 required field처럼 표현력이 필요한 검증을 담당합니다. DB의 `SourceConfiguration` constraint는 세 historical dataset과 publication mode, host/path, HTTPS·GET·auth mode 조합을 고정해 ORM 이외의 쓰기나 회귀도 막습니다. 둘은 중복이라기보다 서로 다른 우회 경로와 오류 범위를 막는 계층입니다.
- secret을 equality/hash/repr에서 분리하는 객체 설계
  - 모범답변: service key는 client나 `PreparedHistoricalRequest`의 필드가 아니라 `build`/fetch 호출의 일시적 인자로만 전달됩니다. 준비 객체는 검증된 semantic query, 값 없는 request shape, scope hash만 보유하고 query 자체도 repr에서 숨깁니다. 따라서 객체 비교·hash·디버그 표현에 credential이 들어갈 저장 지점 자체가 없습니다.
- hash versioning이 필요한 이유
  - 모범답변: canonical field 집합이나 정렬·직렬화 규칙이 바뀌면 같은 의미의 요청도 다른 hash가 될 수 있고, 반대로 새 의미 필드를 누락하면 다른 요청이 같은 scope로 보일 수 있습니다. format/revision을 입력에 포함하면 어느 규칙으로 만든 hash인지 구분해 점진적으로 이행할 수 있습니다. 현재 원본 scope는 dataset과 정렬된 conditions를 hash하므로, 명시적 version을 추가하려면 기존 hash와의 migration도 함께 설계해야 합니다.

### 원본 확인 위치

- Thread 03, Thread 07, Thread 09
- 커밋 `feat(source): persist historical request scopes`
- 파일 `grocery/source/historical_persistence.py`
- 파일 `grocery/source/historical_client.py`
- 구성 요소 `PreparedHistoricalRequest`, `prepare_historical_request`, `start_historical_fetch`, `HISTORICAL_ENDPOINT_CONTRACTS`

## P10. [Thread 04 / `feat(source): parse kamis recent rows`] 정확한 row 계약, 중복 의미 키, 결정적 결과 hash

**우선순위:** S

### 면접 질문

- 외부 JSON row에서 알 수 없는 field를 무시하지 않고 실패시킨 이유는 무엇인가요?
- 입력 row 순서나 object key 순서가 달라도 같은 parse 결과 hash를 만들려면 무엇을 고정해야 하나요?
- 동일 semantic key에 값만 다른 두 row가 있으면 dedupe할지 실패할지 어떻게 판단했나요?
- 꼬리 질문: 오류 메시지에 잘못된 원문 값을 넣지 않고도 디버깅 가능하게 만드는 방법은 무엇인가요?
  - 모범답변: **프로젝트 특수사항:** `KamisParseError`는 고정 code와 `row_index`, `field`만 문자열에 넣습니다. 실제 테스트도 unknown field에 `secret-value`를 넣은 뒤 예외 문자열에 `secret`이 없는지 확인하므로, 어느 행·필드·검증 단계에서 실패했는지는 알되 값은 노출하지 않습니다. **일반 원칙:** 오류에는 안전한 위치 식별자와 분류 코드만 싣고, 원문 값이나 전체 row는 권한이 분리된 진단 경로가 없는 한 출력하지 않습니다.

### 30초 모범 답변

source schema drift를 조기에 감지하려고 row field 집합과 문자열 타입을 정확히 검사하고 unknown field도 실패시킵니다. 각 row는 승인된 코드-이름-단위 registry와 수치·날짜 invariant를 통과한 뒤 semantic key로 정렬하고 canonical data만 hash합니다. 같은 semantic key가 두 번 나오면 어느 값을 선택할 근거가 없으므로 중복으로 실패하며, 오류는 row index·field·고정 코드만 남겨 원문 노출을 막습니다.

### 답변 핵심 키워드

`exact schema`, `fail-closed parser`, `semantic key`, `stable sort`, `canonical hash`, `redacted error`

### 백지 구현

**구현 목표**

정확한 field 집합을 가진 외부 row 목록을 typed row로 파싱하고, 입력 순서와 무관한 결정적 결과 hash를 생성한다.

**인터페이스 또는 함수 시그니처**

```python
def parse_rows(
    items: object,
    *,
    expected_fields: frozenset[str],
    registry: IdentityRegistry,
) -> ParsedResult:
    import hashlib
    import json
    from collections.abc import Mapping, Sequence

    if not isinstance(items, Sequence) or isinstance(items, (str, bytes, bytearray)):
        raise KamisParseError("items_not_array")

    parsed_rows = []
    out_of_scope_row_hashes = []
    seen_keys = set()
    for row_index, raw_row in enumerate(items):
        if not isinstance(raw_row, Mapping):
            raise KamisParseError("row_not_object", row_index=row_index)
        if not all(isinstance(key, str) for key in raw_row):
            raise KamisParseError("non_string_field_name", row_index=row_index)
        actual_fields = frozenset(raw_row)
        if actual_fields != expected_fields:
            if missing := expected_fields - actual_fields:
                raise KamisParseError(
                    "missing_field", row_index=row_index, field=sorted(missing)[0]
                )
            raise KamisParseError("unknown_field", row_index=row_index)
        row = {str(key): value for key, value in raw_row.items()}

        # recent 계약은 reference price의 None만 허용하며, target 밖 row도 타입 검증 후 hash로 남긴다.
        if not _is_target_scope(row, identity_registry=registry):
            _validate_contract_types(row, row_index=row_index)
            out_of_scope_row_hashes.append(_canonical_hash(row))
            continue

        parsed = _parse_row(
            row,
            row_index=row_index,
            identity_registry=registry,
        )
        semantic_key = parsed.semantic_identity_key
        if semantic_key in seen_keys:
            raise KamisParseError("duplicate_semantic_identity", row_index=row_index)
        seen_keys.add(semantic_key)
        parsed_rows.append(parsed)

    ordered_rows = tuple(sorted(parsed_rows, key=lambda row: row.semantic_identity_key))
    ordered_out_of_scope_hashes = tuple(sorted(out_of_scope_row_hashes))
    canonical = json.dumps(
        {
            "accepted_rows": [row.canonical_data() for row in ordered_rows],
            "out_of_scope_row_hashes": ordered_out_of_scope_hashes,
            "parser_contract": "kamis-recent-price-v1",
        },
        ensure_ascii=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")
    return ParsedResult(
        rows=ordered_rows,
        input_row_count=len(items),
        out_of_scope_row_count=len(ordered_out_of_scope_hashes),
        out_of_scope_row_hashes=ordered_out_of_scope_hashes,
        result_hash=hashlib.sha256(canonical).hexdigest(),
    )
```

**입력과 출력**

- 입력: JSON에서 얻은 임의 객체와 승인된 exact field set/identity registry
- 출력: semantic key 순서의 immutable row tuple, input count, result hash

**반드시 만족해야 할 조건**

- items는 sequence이고 각 row는 string key mapping이어야 한다.
- field 집합은 expected set과 정확히 같아야 한다.
- 각 field 값은 승인된 문자열 타입과 bounded 형식을 만족해야 한다.
- identity code/name/unit는 registry와 정확히 일치해야 한다.
- semantic key 중복을 거부한다.
- canonical row를 semantic key로 정렬한 뒤 contract version과 함께 hash한다.
- 오류에는 원문 값이 아니라 code, row index, field만 포함한다.

**경계 조건**

- 빈 목록의 허용 여부를 계약으로 명시하는 경우
- 입력 row 순서와 mapping key 순서가 반대인 경우
- 한글·선행 0 코드·두 자리 소수
- 같은 key와 완전히 같은 row가 중복된 경우도 실패하는 정책

**실패 조건**

- missing/unknown field
- 비문자 field name 또는 value type drift
- 코드-이름/단위 drift
- 중복 semantic identity
- 날짜·Decimal·범위 invariant 위반

**제약**

- unknown field를 자동 보존하거나 무시하지 않는다.
- 원문 row 전체를 예외 문자열로 출력하지 않는다.
- 25~30분 이내 구현할 수 있도록 field별 세부 parser는 제공된다고 가정한다.

### 구현 후 자가 검증

- [ ] row 순서를 바꿔도 결과와 hash가 같다.
- [ ] object key 순서를 바꿔도 결과와 hash가 같다.
- [ ] 한 field 값이 바뀌면 hash가 바뀐다.
- [ ] 중복 semantic key가 검출된다.
- [ ] 오류 문자열에 private marker가 없다.
- [ ] 입력 list와 dict를 수정하지 않는다.

### 구현 후 설명할 것

- 관대한 parser보다 strict parser를 택한 이유
  - 모범답변: 외부 API의 추가·누락 field나 타입 변화는 데이터가 조금 늘어난 것이 아니라 우리가 해석하는 계약이 바뀐 신호일 수 있습니다. 원본은 exact field set, 문자열/nullable 규칙, 날짜·Decimal 범위, code-name-unit registry를 모두 검사해 drift를 즉시 실패시킵니다. 조용히 무시하면 잘못 해석한 값이 정상 publication까지 전파될 수 있습니다.
- canonicalization 단계와 validation 단계를 분리한 이유
  - 모범답변: validation은 source row를 신뢰 가능한 typed 값으로 바꾸며 형식·범위·identity invariant를 확인합니다. canonicalization은 그 검증된 값만 semantic key로 정렬하고 key 순서를 고정해 hash를 만듭니다. 두 단계를 분리하면 비정상 원문이 hash 계약에 섞이지 않고, 입력 배열이나 JSON object 순서가 달라도 같은 의미 결과를 얻습니다.
- 중복을 임의 선택하지 않고 failure로 보내는 이유
  - 모범답변: 같은 semantic key가 둘이면 어느 row가 권위 있는지 source 계약만으로 결정할 근거가 없습니다. 값이 다르면 임의 선택이 사실을 바꾸고, 완전히 같아도 count와 source 품질 문제를 숨깁니다. 그래서 원본의 recent·monthly·regional·market parser는 모두 첫 중복에서 `duplicate_semantic_identity`로 실패합니다.
- 결정적 hash가 재현·승인·publication에 연결되는 방식
  - 모범답변: parser는 검증된 row를 semantic key로 정렬하고 parser contract와 함께 hash합니다. `ParseRun`은 artifact·parser revision·configuration hash에 이 result hash를 저장해 replay가 같은지 확인하고, review와 publication은 `VALIDATED` run의 typed facts를 대상으로 진행합니다. 따라서 같은 증거와 계약에서 다른 hash가 나오면 publication 전에 비결정성을 차단할 수 있습니다.

### 원본 확인 위치

- Thread 04, Thread 07
- 커밋 `feat(source): parse kamis recent rows`
- 커밋 `feat(source): validate historical row primitives`
- 파일 `grocery/source/historical_parser.py`
- 구성 요소 `parse_recent_price_rows`, `HistoricalRowValidator`, `parse_monthly_price_rows`, `parse_regional_price_rows`, `parse_market_price_rows`

## P11. [Thread 04 / `feat(source): seal reviewed series allowlist`] 검토된 identity/dimension registry와 drift 차단

**우선순위:** A

### 면접 질문

- 외부 API가 새로운 품목 코드나 이름을 보내면 자동 등록하지 않고 실패시킨 이유는 무엇인가요?
- registry를 immutable하게 만들고 evidence revision을 함께 둔 이유는 무엇인가요?
- 시장 코드는 region과 독립 키가 아니라 `(region, market)` 쌍으로 검증해야 하는 이유는 무엇인가요?
- 꼬리 질문: 운영 중 registry 업데이트가 필요할 때 데이터 수집과 publication을 어떻게 분리하겠습니까?
  - 모범답변: **프로젝트 특수사항:** 현재 parser는 승인 registry 밖 identity를 자동 수집하지 않고 실패하며 원문도 보존하지 않습니다. 따라서 codebook·unit 근거를 사람이 검토해 새 evidence/configuration revision을 만든 뒤 다시 수집·파싱하고, 기존과 별개의 `ParseRun`을 review한 후에만 publication 대상으로 삼아야 합니다. **일반 원칙:** 새 identity는 격리된 staging에서 관찰할 수 있어도 검토 전에는 공개 fact set에 합치지 않고, 승인된 version 경계를 통해서만 승격합니다.

### 30초 모범 답변

새 코드나 이름은 단순 데이터가 아니라 의미 계약 변경이므로 자동 수용하지 않습니다. 승인된 registry는 복사 후 immutable하게 고정하고 evidence revision을 남겨 어떤 codebook을 근거로 파싱했는지 추적합니다. region·market 관계와 품목 코드-이름-단위 조합을 정확히 검증하며, drift는 별도 사람 검토와 새 revision 후에만 수용합니다.

### 답변 핵심 키워드

`reviewed registry`, `immutability`, `evidence revision`, `code-name drift`, `region-market pair`, `human gate`

### 백지 구현

**구현 목표**

승인된 품목·지역·시장 registry에 대해 관측 identity가 정확히 일치하는지 검증한다.

**인터페이스 또는 함수 시그니처**

```python
def validate_observation(
    registry: ReviewedRegistry,
    observation: SourceObservation,
) -> ValidatedIdentity:
    if not registry.dimension_evidence_revision.strip():
        raise ValueError("dimension_evidence_revision is required")

    identity = IdentityObservation(
        product_class_code=observation.product_class_code,
        product_class_name=observation.product_class_name,
        category_code=observation.category_code,
        category_name=observation.category_name,
        item_code=observation.item_code,
        item_name=observation.item_name,
        variety_code=observation.variety_code,
        variety_name=observation.variety_name,
        grade_code=observation.grade_code,
        grade_name=observation.grade_name,
        raw_unit=observation.raw_unit,
        raw_unit_size=observation.raw_unit_size,
        coverage_identity=registry.identity_registry.coverage_identity,
    )
    # code-name-grade-unit 계약은 live row가 아니라 독립 검토된 registry가 결정한다.
    registry.identity_registry.validate(identity, row_index=0)

    expected_region_name = registry.region_names.get(observation.region_code)
    if expected_region_name != observation.region_name:
        raise KamisParseError("region_code_name_drift", row_index=0)

    market_key = (observation.region_code, observation.market_code)
    expected_market_name = registry.market_names.get(market_key)
    if expected_market_name != observation.market_name:
        raise KamisParseError("market_code_name_drift", row_index=0)

    return ValidatedIdentity(
        identity=identity,
        region=RegionObservation(
            code=observation.region_code,
            name=observation.region_name,
        ),
        market=MarketObservation(
            code=observation.market_code,
            name=observation.market_name,
        ),
        evidence_revision=registry.dimension_evidence_revision,
    )
```

**입력과 출력**

- 입력: immutable reviewed registry와 source observation
- 출력: 검증된 identity 값 객체

**반드시 만족해야 할 조건**

- 품목 계열의 코드-이름-등급-원문 단위가 exact contract와 일치해야 한다.
- region code-name이 exact mapping과 일치해야 한다.
- market은 `(region_code, market_code)` mapping으로 검증한다.
- unknown code를 registry에 자동 추가하지 않는다.
- registry 생성 후 원본 dict 변경이 내부 상태에 영향을 주지 않아야 한다.

**경계 조건**

- 같은 시장 코드가 다른 region에 존재할 수 있는 경우
- 선행 0 코드
- Unicode 제어 문자가 포함된 이름
- display name만 drift한 경우

**실패 조건**

- unknown code
- code-name mismatch
- region-market 관계 불일치
- 빈 evidence revision
- registry 외부 mutation 가능

**제약**

- fuzzy matching이나 문자열 유사도로 대체하지 않는다.
- registry 업데이트 기능은 구현하지 않는다.
- 15~20분 구현 크기다.

### 구현 후 자가 검증

- [ ] 원본 mapping을 생성 후 변경해도 registry가 변하지 않는다.
- [ ] 선행 0 코드가 유지된다.
- [ ] market이 잘못된 region 아래에서 통과하지 않는다.
- [ ] 이름 drift가 unknown이 아닌 명확한 mismatch로 구분된다.
- [ ] 오류에 원문 전체 row가 노출되지 않는다.

### 구현 후 설명할 것

- 자동 schema evolution을 거부한 이유
  - 모범답변: provider가 보낸 새 code나 이름은 그 자체로 의미의 권위가 될 수 없습니다. 자동 등록하면 오타, 재사용된 code, 단위 변경도 정상 schema evolution으로 굳어져 publication 의미가 조용히 바뀝니다. 프로젝트는 live 응답과 독립적으로 검토한 codebook·unit evidence만 registry에 넣고, 관측 drift는 실패시킵니다.
- immutable registry와 evidence revision의 감사 가치
  - 모범답변: `ExactIdentityRegistry`와 `HistoricalDimensionRegistry`는 입력 mapping을 복사한 뒤 `MappingProxyType`으로 감싸므로 생성 뒤 원본 dict 변경이 parse 의미를 바꾸지 않습니다. codebook hash, unit-contract hash, coverage/dimension evidence revision을 함께 남기면 어떤 검토 근거와 계약으로 row를 승인했는지 재현할 수 있습니다.
- identity drift와 단순 display 변경을 구분하는 기준
  - 모범답변: 이 프로젝트에서는 code뿐 아니라 code-name과 원문 unit 조합도 승인된 identity 계약입니다. 따라서 이름만 달라져도 자동으로 “display 변경”이라 낮춰 보지 않고 명시적인 `*_code_name_drift`로 실패합니다. 정말 표시명 변경이라면 근거를 검토해 registry revision을 올리는 것이 둘을 구분하는 checkpoint입니다.
- 사람 검토 checkpoint가 ingestion throughput에 주는 trade-off
  - 모범답변: 새 품목이나 이름이 나타날 때 자동 적재가 멈추므로 처리량과 최신성은 떨어지고 재검토·재수집 비용이 생깁니다. 대신 잘못된 identity와 단위가 대량의 typed fact 및 publication으로 번지는 것을 막고, 승인한 변경마다 근거 revision을 남길 수 있습니다.

### 원본 확인 위치

- Thread 04, Thread 07, Thread 08
- 커밋 `feat(source): seal reviewed series allowlist`
- 파일 `grocery/source/registry.py`
- 구성 요소 `INITIAL_RETAIL_IDENTITY_REGISTRY`, `OFFICIAL_DOCS_ZIP_SHA256`, `HistoricalDimensionRegistry`

## P12. [Thread 03 / `feat(source): share bounded transport for history`] bounded pagination과 재시도 폭주 방지

**우선순위:** A

### 면접 질문

- 외부 API pagination에서 page 수, row 수, byte 수를 모두 제한해야 하는 이유는 무엇인가요?
- provider의 total count가 페이지마다 달라질 때 마지막 값을 믿지 않고 실패시키는 이유는 무엇인가요?
- retry를 transport 내부에서 무제한 수행하지 않은 이유는 무엇인가요?
- 꼬리 질문: 부분 receipt를 유지하면서도 중복 row를 막는 semantic boundary는 어디인가요?
  - 모범답변: **프로젝트 특수사항:** transport receipt는 page의 순서·count·byte·hash만 증명하며 row identity를 판정하지 않습니다. 중복 의미 row는 그 다음 strict parser가 dataset별 semantic key를 만든 뒤 `duplicate_semantic_identity`로 차단합니다. **일반 원칙:** 전송 계층은 임의 dedupe로 원문 의미를 바꾸지 말고, domain key를 아는 parser/ingestion 경계가 중복 정책을 소유해야 합니다.

### 30초 모범 답변

외부 total이나 body 크기는 신뢰 경계 밖이므로 page·row·byte·timeout을 각각 제한해야 메모리와 실행 시간을 예측할 수 있습니다. 페이지 번호와 declared total이 일관되고 누적 row가 정확히 맞을 때만 성공하며, drift는 임의 보정하지 않습니다. transport는 bounded 시도와 partial receipt까지만 책임지고, 재시도 정책은 scheduler나 호출자가 결정해 중첩 retry와 폭주를 막습니다.

### 답변 핵심 키워드

`bounded I/O`, `pagination reconciliation`, `timeout`, `retry ownership`, `partial receipt`, `resource budget`

### 백지 구현

**구현 목표**

주어진 `fetch_page` 콜백으로 bounded pagination을 수행하고, 성공 또는 안전한 partial failure 정보를 반환한다.

**인터페이스 또는 함수 시그니처**

```python
def fetch_all_pages(
    fetch_page: FetchPage,
    *,
    max_pages: int,
    max_rows: int,
    max_bytes: int,
) -> FetchResult:
    import hashlib
    import json
    import re

    for name, value in (
        ("max_pages", max_pages),
        ("max_rows", max_rows),
        ("max_bytes", max_bytes),
    ):
        if not isinstance(value, int) or isinstance(value, bool) or value < 1:
            raise TransportFailure(f"invalid_{name}")

    rows = []
    receipts = []
    expected_total = None
    expected_page_size = None
    received_bytes = 0
    page_number = 1

    try:
        while True:
            if page_number > max_pages:
                raise TransportFailure("page_budget_exceeded", page_number=page_number)

            page = fetch_page(page_number)
            if page.http_status != 200:
                raise TransportFailure(
                    "terminal_http_status",
                    page_number=page_number,
                    http_status=page.http_status,
                )
            if page.provider_result_code != "0":
                raise TransportFailure(
                    "terminal_provider_error",
                    page_number=page_number,
                    provider_result_code=page.provider_result_code,
                )
            if page.declared_page_number != page_number:
                raise TransportFailure("declared_page_mismatch", page_number=page_number)
            if (
                not isinstance(page.declared_page_size, int)
                or isinstance(page.declared_page_size, bool)
                or page.declared_page_size < 1
            ):
                raise TransportFailure("invalid_declared_page_size", page_number=page_number)
            if (
                not isinstance(page.declared_total_count, int)
                or isinstance(page.declared_total_count, bool)
                or page.declared_total_count < 0
            ):
                raise TransportFailure("invalid_declared_total", page_number=page_number)
            if (
                not isinstance(page.byte_length, int)
                or isinstance(page.byte_length, bool)
                or page.byte_length <= 0
                or not isinstance(page.body_sha256, str)
                or re.fullmatch(r"[0-9a-f]{64}", page.body_sha256) is None
            ):
                raise TransportFailure("invalid_response_state", page_number=page_number)

            if expected_total is None:
                expected_total = page.declared_total_count
                expected_page_size = page.declared_page_size
                required_pages = max(
                    1,
                    (expected_total + expected_page_size - 1) // expected_page_size,
                )
                if required_pages > max_pages:
                    raise TransportFailure("page_budget_exceeded", page_number=page_number)
                if expected_total > max_rows:
                    raise TransportFailure("row_budget_exceeded", page_number=page_number)
            elif page.declared_total_count != expected_total:
                raise TransportFailure("declared_total_changed", page_number=page_number)
            elif page.declared_page_size != expected_page_size:
                raise TransportFailure("declared_page_size_mismatch", page_number=page_number)

            remaining = expected_total - len(rows)
            expected_count = min(expected_page_size, max(0, remaining))
            if len(page.items) != expected_count:
                raise TransportFailure("page_row_count_mismatch", page_number=page_number)

            next_row_count = len(rows) + len(page.items)
            next_byte_count = received_bytes + page.byte_length
            # 상한을 넘긴 page는 완료 receipt로 인정하기 전에 실패시킨다.
            if next_row_count > max_rows:
                raise TransportFailure("row_budget_exceeded", page_number=page_number)
            if next_byte_count > max_bytes:
                raise TransportFailure("byte_budget_exceeded", page_number=page_number)

            receipt = PageReceipt(
                ordinal=len(receipts) + 1,
                requested_page_number=page_number,
                declared_page_number=page.declared_page_number,
                declared_page_size=page.declared_page_size,
                declared_total_count=page.declared_total_count,
                row_count=len(page.items),
                http_status=page.http_status,
                provider_result_code=page.provider_result_code,
                byte_length=page.byte_length,
                body_sha256=page.body_sha256,
            )
            rows.extend(page.items)
            receipts.append(receipt)
            received_bytes = next_byte_count

            if len(rows) == expected_total:
                break
            if len(rows) > expected_total:
                raise TransportFailure("row_total_exceeded", page_number=page_number)
            page_number += 1
    except TransportFailure as error:
        # 원본과 같이 실패한 page는 제외하고 완전히 끝난 prefix만 전달한다.
        error._retain_completed_pages(tuple(receipts))
        raise

    frozen_receipts = tuple(receipts)
    manifest = json.dumps(
        [receipt.body_sha256 for receipt in frozen_receipts],
        ensure_ascii=True,
        separators=(",", ":"),
    ).encode("ascii")
    return FetchResult(
        rows=tuple(rows),
        page_receipts=frozen_receipts,
        ordered_manifest_sha256=hashlib.sha256(manifest).hexdigest(),
        call_count=len(frozen_receipts),
    )
```

**입력과 출력**

- 입력: page number를 받아 page result를 반환하는 콜백과 자원 상한
- 출력: 연속 page receipts와 rows, 또는 partial receipts를 가진 bounded failure

**반드시 만족해야 할 조건**

- page 1부터 순서대로 요청한다.
- 각 응답의 declared page/total과 HTTP/provider status를 검증한다.
- page·row·byte 상한을 넘기기 전에 중단한다.
- 누적 row 수가 declared total에 도달하면 종료하고 초과도 실패한다.
- 실패 시 완료된 page receipt를 순서대로 보존한다.
- 내부에서 무제한 retry하지 않는다.

**경계 조건**

- total=0 정책
- 마지막 페이지가 비어 있는 provider 동작
- 정확히 상한에 도달하는 응답
- 첫 페이지 후 total이 바뀌는 응답

**실패 조건**

- page number 불일치
- HTTP/provider 실패
- total drift
- 예산 초과
- timeout/network exception

**제약**

- 동시에 여러 page를 speculative fetch하지 않는다.
- 응답 원문을 오류 문자열에 포함하지 않는다.
- 20~25분 구현 크기다.

### 구현 후 자가 검증

- [ ] 한 페이지와 여러 페이지가 모두 정상 종료된다.
- [ ] 정확한 상한과 상한+1이 구분된다.
- [ ] partial failure에 완료된 receipt만 들어 있다.
- [ ] total drift를 마지막 값으로 덮지 않는다.
- [ ] 호출 횟수가 max_pages를 넘지 않는다.

### 구현 후 설명할 것

- 다중 자원 budget을 둔 이유
  - 모범답변: page/call 수는 quota와 실행 시간을, row 수는 후속 parse 메모리·작업량을, byte 수는 네트워크와 메모리 사용을 각각 제한합니다. 한 축만 제한하면 작은 row를 매우 많이 받거나 적은 row의 거대한 body를 받는 식으로 다른 자원이 폭주할 수 있습니다. 원본도 call·page·page-byte·timeout을 독립 경계로 둡니다.
- retry 책임을 scheduler로 올린 이유
  - 모범답변: 원본 transport는 transient HTTP/provider/network 오류만 page당 최대 3회, 전체 call budget 안에서 재시도합니다. 그 이상인 전체 attempt 재시도는 scheduler/호출자가 맡아 backoff와 실행 횟수를 한곳에서 통제합니다. 계층마다 자체 retry를 중첩하면 실제 호출 수가 곱으로 늘고 quota 고갈이나 retry storm이 생깁니다.
- 순차 pagination의 단순성 대 병렬 처리 성능 trade-off
  - 모범답변: 순차 요청은 첫 total을 기준으로 다음 page의 번호·크기·예상 row 수를 즉시 대사하고, 실패 시 완료된 prefix를 정확히 남기기 쉽습니다. speculative 병렬 fetch보다 지연 시간은 길지만, total drift와 budget 초과 때 이미 발사된 불필요한 요청이 없고 ordered manifest도 단순합니다.
- provider total을 consistency signal로 사용하는 방법
  - 모범답변: 첫 page의 declared total을 고정하고 이후 모든 page가 같은 값을 선언하는지 확인합니다. 매 page에서는 남은 row와 page size로 정확한 예상 row 수를 계산하고, 누적 row가 total과 같을 때만 성공합니다. total 변화, 초과, 이른 빈 page는 보정하지 않고 reconciliation failure로 처리합니다.

### 원본 확인 위치

- Thread 03
- 커밋 `feat(source): share bounded transport for history`
- 파일 `grocery/source/client.py`
- 구성 요소 `KamisHttpClient.fetch_historical_prices`, `_fetch_prices`
- 연관 Thread 20, 21

## P13. [Thread 04 / `feat(audit): reconcile deterministic parses`] artifact·parse run 상태 머신과 count reconciliation

**우선순위:** A

### 면접 질문

- 왜 artifact를 원문 payload가 아니라 ordered manifest hash와 크기 메타데이터로 모델링했나요?
- parse run의 idempotency key에 artifact, parser revision, configuration hash가 모두 필요한 이유는 무엇인가요?
- `total = accepted + out_of_scope + quarantined` 같은 count invariant는 어떤 오류를 잡나요?
- 꼬리 질문: 같은 입력에서 result hash가 달라지면 retry할지 quarantine할지 어떻게 판단합니까?
  - 모범답변: **프로젝트 특수사항:** 이미 `VALIDATED`인 run에 같은 artifact·parser revision·configuration으로 다른 result hash가 오면 `NONDETERMINISTIC_REPLAY`로 실패하고 기존 run과 fact를 보존합니다. 현재 코드는 이를 자동 retry나 quarantine으로 바꾸지 않으므로 운영 조사 후 parser/configuration revision을 분리해야 합니다. **일반 원칙:** transient I/O 실패만 제한적으로 retry하고, 같은 결정적 입력의 결과 drift는 반복 실행으로 덮지 말고 격리·경보·원인 분석 대상으로 취급합니다.

### 30초 모범 답변

artifact는 수집된 페이지들의 순서·hash·길이를 묶은 불변 증거로 두고 원문은 보존하지 않았습니다. parse run은 같은 artifact·parser revision·configuration에서 재실행 가능한 단위이며, 동일 key의 result hash가 달라지면 비결정성으로 취급해야 합니다. 상태와 count 합계, missing reference가 accepted 안에 포함된다는 invariant를 DB까지 내려 부분 집계나 잘못된 완료를 막습니다.

### 답변 핵심 키워드

`hash-only artifact`, `parse idempotency`, `configuration hash`, `count reconciliation`, `nondeterminism`, `quarantine`

### 백지 구현

**구현 목표**

artifact와 parser/config 버전을 키로 parse run을 시작하거나 replay하고, count 및 result hash를 검증해 완료한다.

**인터페이스 또는 함수 시그니처**

```python
def complete_parse_run(
    run: ParseRunState,
    result: ParsedGeneration,
) -> ParseRunState:
    import re
    from dataclasses import replace
    from datetime import datetime, timezone

    counts = (
        result.total_row_count,
        result.accepted_row_count,
        result.out_of_scope_row_count,
        result.quarantined_row_count,
        result.missing_reference_row_count,
    )
    if any(
        not isinstance(count, int) or isinstance(count, bool) or count < 0
        for count in counts
    ):
        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)
    if result.total_row_count != (
        result.accepted_row_count
        + result.out_of_scope_row_count
        + result.quarantined_row_count
    ):
        raise ParseGenerationError(
            ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED
        )
    if result.missing_reference_row_count > result.accepted_row_count:
        raise ParseGenerationError(
            ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED
        )
    if result.quarantined_row_count != 0:
        # 원본 상태 계약에서 quarantine이 남은 run은 VALIDATED로 완료될 수 없다.
        raise ParseGenerationError(
            ParseGenerationErrorCode.RESULT_RECONCILIATION_FAILED
        )
    if (
        not isinstance(result.result_hash, str)
        or re.fullmatch(r"[0-9a-f]{64}", result.result_hash) is None
    ):
        raise ParseGenerationError(ParseGenerationErrorCode.RESULT_CONTRACT_INVALID)

    expected_outcome = {
        "result_hash": result.result_hash,
        "total_row_count": result.total_row_count,
        "accepted_row_count": result.accepted_row_count,
        "missing_reference_row_count": result.missing_reference_row_count,
        "out_of_scope_row_count": result.out_of_scope_row_count,
        "quarantined_row_count": result.quarantined_row_count,
        "failure_code": "",
    }

    if run.status == "VALIDATED":
        if any(
            getattr(run, field) != expected
            for field, expected in expected_outcome.items()
        ):
            raise ParseGenerationError(ParseGenerationErrorCode.NONDETERMINISTIC_REPLAY)
        return run
    if run.status != "STARTED":
        raise ParseGenerationError(ParseGenerationErrorCode.PARSE_RUN_NOT_STARTED)

    # artifact/parser/config identity는 바꾸지 않고 outcome만 한 번 채운다.
    return replace(
        run,
        status="VALIDATED",
        completed_at=datetime.now(timezone.utc),
        **expected_outcome,
    )
```

**입력과 출력**

- 입력: STARTED parse run과 count/result hash를 가진 parse 결과
- 출력: VALIDATED 또는 명시적 실패 상태의 parse run

**반드시 만족해야 할 조건**

- total, accepted, out_of_scope, quarantined는 음수가 아니다.
- `total == accepted + out_of_scope + quarantined`를 만족한다.
- missing reference count는 accepted 이하이다.
- result hash는 canonical lowercase SHA-256이다.
- 완료 replay는 count와 hash가 완전히 같을 때만 허용한다.
- 동일 idempotency key의 다른 결과는 비결정성 충돌로 처리한다.

**경계 조건**

- accepted=0인 검증 결과
- 모든 row가 out of scope인 경우
- 동일 결과 replay
- parser revision만 바뀐 재실행

**실패 조건**

- count 합계 불일치
- invalid status transition
- 잘못된 result hash
- 동일 key의 결과 drift

**제약**

- 원문 payload를 parse run에 저장하지 않는다.
- 상태 변경과 count 저장은 원자적이어야 한다.
- 20분 내 구현한다.

### 구현 후 자가 검증

- [ ] 정상 count 조합이 완료된다.
- [ ] 합계가 1만 달라도 실패한다.
- [ ] 동일 replay가 새 run을 만들지 않는다.
- [ ] parser/config 버전 변화는 다른 run으로 구분된다.
- [ ] result hash drift가 기존 run을 덮어쓰지 않는다.

### 구현 후 설명할 것

- artifact와 parse run을 분리한 이유
  - 모범답변: artifact는 수집 page들의 hash·순서·크기를 나타내는 parser 독립적 입력 증거이고, parse run은 그 artifact를 특정 parser revision과 configuration으로 해석한 결과입니다. 분리하면 같은 수집 증거를 새 parser/config로 다시 해석해도 원본 수집 identity를 복제하거나 덮어쓰지 않고 결과 계보를 각각 남길 수 있습니다.
- configuration hash가 재현성에 필요한 이유
  - 모범답변: 같은 코드 revision이라도 reviewed registry의 이름·unit·evidence나 source contract 설정이 달라지면 parse 결과가 달라질 수 있습니다. 원본은 이 값들을 canonical하게 hash해 artifact·parser revision과 함께 unique key로 사용하므로, 숨은 설정 변경을 동일 run의 replay로 오인하지 않습니다.
- count invariant를 DB에 둘 가치
  - 모범답변: 서비스는 완료 전에 `total = accepted + out_of_scope + quarantined`와 missing-reference가 accepted 이하인지 검사해 명확한 오류를 낼 수 있습니다. 같은 식을 DB constraint에도 두면 다른 쓰기 경로, 코드 회귀, 동시성 또는 부분 업데이트가 잘못된 완료 집계를 저장하는 것을 최종적으로 거부합니다.
- 비결정성 발견 시 fail-closed 정책의 장단점
  - 모범답변: 장점은 먼저 저장된 validated 결과를 새 결과로 덮지 않고 publication에 어느 결과가 맞는지 모르는 상태가 섞이는 것을 막는 점입니다. 단점은 작은 비결정적 요소도 자동 복구하지 않아 가용성이 낮아지고 수동 조사와 새 revision이 필요하다는 점입니다. 감사 가능한 가격 사실에서는 이 비용보다 모호한 결과 공개 위험을 더 크게 본 설계입니다.

### 원본 확인 위치

- Thread 04
- 커밋 `feat(audit): reconcile deterministic parses`
- 파일 `grocery/models.py`
- 구성 요소 `SourceArtifact`, `ParseRun`, `build_source_artifact`, `start_or_get_kamis_parse_run`, `complete_kamis_parse_generation`
