# 소요시간 도출과 후보 무결성 워크북

이 문서는 공식 운항 기록을 결정적인 route-duration evidence로 축약하는 알고리즘과, 그렇게 만든 여러 타입화된 row를 “검토 가능한 하나의 후보 집합”으로 봉인하는 aggregate integrity 설계를 다룬다.

<a id="w08"></a>

---

# [travel_windows/duration_reference.py::_derive_direction] W08 — 공식 운항 자료에서 강건한 양방향 소요시간 도출 (S)

## 면접 질문

`_consume_csv`는 동일 운항 occurrence에 여러 duration 값이 들어올 수 있게 모은 뒤 occurrence별 upper median을 하나만 만들고, `_derive_direction`에서 다시 MAD 기반 이상치 제거와 IQR 교차검증을 수행합니다. 단순 전체 평균이나 전체 median 대신 이 두 단계 축약을 선택한 이유를 설명해 보세요.

꼬리 질문:

1. occurrence identity에 운항일·양쪽 예정일·항공기 등록번호를 포함하는 이유는 무엇인가요?
2. 최소 5개 occurrence와 5개 서로 다른 날짜를 모두 요구하는 이유는 무엇인가요?
3. MAD가 0일 때 최소 30분 threshold를 두는 효과와 trade-off는 무엇인가요?
4. 표본이 8개 이상일 때 IQR 결과와 최종 upper median이 15분 넘게 다르면 전체를 conflict로 거부하는 이유는 무엇인가요?

## 30초 모범 답변

원본 CSV에는 같은 실제 운항을 여러 행이 중복 관찰할 수 있어 행 수가 많은 occurrence가 결과를 과도하게 지배할 수 있습니다. 먼저 occurrence별 upper median으로 한 표를 만들고, 최소 표본·날짜 분산을 확인한 뒤 median/MAD로 이상치를 제거합니다. 적어도 5개와 전체의 70%가 남아야 하며, 큰 표본은 IQR 방식과도 결과가 가까운지 확인합니다. 이로써 결과는 결정적이고 이상치에 강하면서, 불안정한 자료는 억지 숫자 대신 폐쇄형 conflict로 격리됩니다.

## 답변 핵심 키워드

- occurrence deduplication
- upper median
- median absolute deviation
- retained-ratio invariant
- IQR cross-check
- bidirectional completeness

## 꼬리 질문과 핵심 답변

### upper median은 일반 median과 어떻게 다르고 왜 보수적인가요?

짝수 개일 때 가운데 두 값의 평균이 아니라 큰 쪽 가운데 값을 선택합니다. 정수 minute를 유지하고 여행 가능 체류시간을 과대평가하지 않도록 소요시간을 약간 보수적으로 잡는 효과가 있습니다.

### 전체 행에서 바로 MAD를 계산하면 무엇이 잘못될 수 있나요?

한 occurrence가 여러 중복 행을 가지면 그 운항의 값이 통계에서 여러 표를 얻습니다. provider 데이터의 중복 정도가 route duration을 바꾸게 되므로 먼저 실제 occurrence 단위로 동등한 한 표를 만들어야 합니다.

### 평균 대신 median 계열을 쓰는 이유는 무엇인가요?

지연·기록 오류처럼 꼬리가 긴 값에 평균은 민감합니다. median과 MAD는 일부 극단값이 있어도 중심과 산포가 안정적입니다. 대신 작은 표본에서 민감도가 떨어지므로 최소 표본과 추가 IQR 검증을 둡니다.

### 70% retained 조건은 무엇을 뜻하나요?

결론을 만들기 위해 원자료 대부분을 버리는 상황을 막습니다. 이상치 필터가 수치 하나를 만들 수 있어도, 30% 넘게 배제해야 한다면 자료 자체가 혼합되거나 불안정하다고 보고 conflict로 닫습니다.

### outbound가 충분하고 inbound가 부족하면 한 방향만 게시해도 되나요?

현재 검색은 outbound duration으로 목적지 도착을, inbound duration으로 ICN 귀환을 계산하므로 둘 중 하나가 없으면 같은 route evidence가 완전하지 않습니다. 전체 route를 격리해야 합니다.

## 백지 구현

### 구현 목표

한 방향의 occurrence별 duration 관측으로부터 강건하고 결정적인 minute 값을 도출한다. 불충분하거나 서로 충돌하는 표본은 숫자를 만들지 않고 고정 실패 code를 반환한다.

### 인터페이스

`derive_direction(occurrences: Mapping[OccurrenceId, Sequence[int]], counts: SourceCounts) -> DirectionResult`

`DirectionResult`는 성공 시 `minutes`, source/invalid/cancelled/occurrence/outlier/retained count를, 실패 시 `INSUFFICIENT_SAMPLE` 또는 `CONFLICTING_SAMPLE`을 가진다.

### 입력

- occurrence ID별 1개 이상의 duration minute 목록
- 각 occurrence ID에 포함된 운항일
- 파싱 단계에서 누적한 source/invalid/cancelled row count
- duration 값은 사전에 허용 범위로 검증되었다고 가정한다.

### 출력

- 성공: 정수 minute와 감사 가능한 count 묶음
- 실패: 숫자 없는 폐쇄형 code

### 반드시 만족해야 할 조건

- 각 occurrence는 값 개수와 무관하게 결과 통계에서 한 표만 가진다.
- occurrence 대표값은 정렬했을 때 index `len // 2`의 upper median이다.
- occurrence와 서로 다른 운항일이 각각 최소 5개여야 한다.
- 중심은 occurrence 대표값의 median이다.
- MAD threshold는 `max(30분, 3 × 1.4826 × MAD)`이다.
- retained sample은 최소 5개이며 원 occurrence의 70% 이상이어야 한다.
- 최종 값은 retained sample의 upper median이다.
- 원 occurrence가 8개 이상이면 Tukey IQR fence로 남긴 값의 upper median과 최종 값 차이가 15분 이하여야 한다.

### 경계 조건

- 정확히 5개 occurrence/5개 날짜
- occurrence 5개지만 날짜는 4개
- 같은 occurrence에 짝수 개의 서로 다른 duration
- 모든 대표값이 같아 MAD가 0
- 정확히 70%가 남는 경우와 그보다 하나 적게 남는 경우
- 7개 표본과 8개 표본의 IQR 교차검증 경계
- IQR이 0인 경우와 fence 경계값과 같은 관측

### 실패 조건

- raw row 전체에 가중치를 주어 중복 occurrence가 결과를 지배함
- 이상치를 많이 버리고도 숫자를 반환함
- outbound/inbound 한쪽 실패를 0 또는 다른 방향 값으로 대체함
- 정렬 tie에서 비결정적 결과를 냄
- 실패 결과에 raw duration 목록이나 파일 row를 포함함

### 필요한 제약사항

- 입력을 수정하지 않는다.
- 정렬 기반 `O(n log n)` 시간과 `O(n)` 추가 공간 이내로 구현한다.
- 평균·floating rounding으로 최종 정수 minute를 임의 생성하지 않는다.

## 구현 후 자가 검증

- [ ] occurrence마다 대표값 하나만 통계에 들어가는가?
- [ ] 짝수 개 표본의 upper median이 큰 가운데 값인가?
- [ ] 최소 occurrence와 최소 날짜 조건을 각각 검사하는가?
- [ ] MAD=0에서도 30분 threshold가 적용되는가?
- [ ] retained 최소 개수와 70% 비율 경계가 정확한가?
- [ ] 8개 이상에서 IQR 교차검증과 15분 경계를 검사하는가?
- [ ] 입력 순서가 달라도 결과와 count가 같은가?
- [ ] 실패 시 raw 표본을 노출하지 않는가?

## 구현 후 설명할 것

- row 단위가 아니라 occurrence 단위로 먼저 축약한 이유
- upper median을 사용한 도메인상 보수성
- MAD와 IQR 두 강건 통계의 역할 차이
- 충분성 조건과 안정성 조건을 분리한 이유
- 숫자를 항상 만들지 않고 conflict로 닫는 운영 trade-off

## 원본 확인 위치

- 파일: `travel_windows/duration_reference.py`
- 함수: `_minutes`, `_consume_csv`, `_upper_median`, `_tukey_quartiles`, `_derive_direction`, `derive_route_durations`
- Commit: `f3075b5 feat(source): collect official route duration reference`
- 관련 테스트: `travel_windows/tests/test_duration_reference.py::DurationReferenceDerivationTests`
- 대표 테스트: `test_derives_bidirectional_upper_medians_and_audit_counts`, `test_rejects_insufficient_and_unstable_direction_samples`
- persistence: `travel_windows/ingestion.py::_persist_duration_derivation`
- 후속 Commit: `5d06f32 feat(duration): seal official route duration derivations`
- publication 재검증: `travel_windows/publication.py::_approved_duration_derivation`

<a id="w09"></a>

---

# [travel_windows/publication.py::_closure_fingerprint, _load_valid_candidate] W09 — 타입화된 flight candidate의 closure seal (S)

## 면접 질문

`stage_flight_evidence`는 schedule revision과 duration membership을 만든 뒤 `FlightCandidateSeal`에 schedule count, duration count, closure fingerprint를 기록하고, `_load_valid_candidate`는 게시 직전에 모든 fingerprint와 count를 다시 계산합니다. 단순히 revision의 `VALIDATED` 상태만 확인하지 않고 aggregate를 봉인하는 이유를 설명해 보세요.

꼬리 질문:

1. closure hash에 schedule fingerprint와 정렬된 `(destination IATA, duration fingerprint)` 목록을 넣는 이유는 무엇인가요?
2. hash가 있는데 count를 별도로 저장·비교하는 이유는 무엇인가요?
3. seal insert와 child membership insert가 동시에 일어나면 어떤 race가 가능하며 PostgreSQL advisory lock trigger가 어떻게 막나요?
4. publish 시 Python 재계산과 DB trigger 검증을 둘 다 두는 이유는 무엇인가요?

## 30초 모범 답변

revision 상태 하나는 검토 대상 child 집합이 이후에도 그대로라는 것을 증명하지 못합니다. 이 후보는 schedule row와 airport별 duration row가 함께 완전해야 하므로, 타입화된 payload hash와 정렬된 membership hash·count로 aggregate closure를 봉인합니다. seal 이후 child append를 DB trigger로 막고, 게시 transaction은 모든 source/parse/duration lineage와 closure를 다시 계산합니다. 같은 advisory key를 seal과 child insert가 잡아 “count한 뒤 새 child가 끼어드는” race도 직렬화합니다.

## 답변 핵심 키워드

- aggregate closure
- candidate seal
- deterministic membership hash
- immutable children
- publish-time revalidation
- advisory-lock race closure

## 꼬리 질문과 핵심 답변

### child row에 각각 immutable trigger가 있으면 seal이 없어도 되지 않나요?

각 row의 값이 안 바뀌는 것과 “검토 시점에 이 집합이 완전했다”는 것은 다릅니다. 새 row append가 가능하면 기존 row는 모두 immutable이어도 aggregate가 바뀝니다. seal은 멤버 집합의 닫힘을 표현합니다.

### 정렬 없이 DB 조회 순서대로 hash하면 어떤 문제가 생기나요?

SQL row 순서는 `ORDER BY` 없이는 보장되지 않습니다. 같은 집합이 실행마다 다른 hash를 만들 수 있으므로 IATA 같은 stable identity로 정렬해야 합니다.

### count가 hash에 이미 암묵적으로 반영되는데 왜 별도 count가 필요한가요?

빠른 구조 검증과 진단에 유용하고, DB trigger가 전체 canonical serializer를 구현하지 않아도 child 수 일치부터 거부할 수 있습니다. hash serialization 버그가 있어도 count가 독립 방어층이 됩니다.

### seal version이 필요한 이유는 무엇인가요?

canonical payload나 migration backfill 방식이 바뀔 수 있습니다. version 없이 같은 hash 필드 의미를 in-place 변경하면 과거 후보를 현재 알고리즘으로 잘못 해석합니다.

### publish 시 이미 seal을 trust하지 않고 source rights와 parse run까지 다시 보는 이유는 무엇인가요?

seal은 typed membership의 동일성을 증명하지만 source 계약의 현재 유효성, artifact/attempt provenance, parse outcome까지 자동으로 증명하지 않습니다. 게시 경계가 최종 소비 권한과 전체 lineage를 다시 확인해야 합니다.

## 백지 구현

### 구현 목표

부모 candidate와 여러 immutable child evidence를 하나의 검토 단위로 봉인하고, 봉인 후 append를 막으며, 게시 직전에 동일 aggregate인지 검증하는 작은 aggregate sealing 모듈을 구현한다.

### 인터페이스

`seal_candidate(candidate_id: UUID, expected_version: str = "V1") -> Seal`

`validate_sealed_candidate(candidate_id: UUID) -> ValidatedCandidate | ValidationFailure`

repository는 candidate/child row lock, candidate별 advisory lock, seal insert, child append를 제공한다고 가정한다.

### 입력

- 하나의 parent schedule revision
- 1개 이상의 schedule row
- 각 공개 destination airport에 정확히 하나의 duration membership
- 각 child의 canonical typed fingerprint

### 출력

- `Seal(version, schedule_count, duration_count, closure_sha256)`
- 검증 성공 시 잠긴 parent와 exact child 목록
- 실패 시 `INCOMPLETE`, `ALREADY_SEALED`, `FINGERPRINT_MISMATCH`, `LINEAGE_INVALID`

### 반드시 만족해야 할 조건

- 모든 destination은 outbound와 inbound schedule을 모두 가져야 한다.
- duration membership의 airport 집합은 schedule destination 집합과 정확히 같아야 한다.
- parent schedule payload와 각 duration payload를 canonical 방식으로 다시 hash한다.
- closure에는 schedule hash와 IATA 기준 정렬된 duration identity 목록이 포함된다.
- seal count는 실제 child count와 일치해야 한다.
- seal 이후 schedule/duration append·update·delete는 거부된다.
- seal과 child append는 같은 candidate lock key로 직렬화된다.
- publication 검증은 저장된 hash를 복사해 비교하지 않고 현재 row에서 재계산한다.

### 경계 조건

- schedule은 있으나 duration이 없음
- 한 destination에 한 방향만 존재
- duration의 airport가 schedule 집합에 없거나 하나 빠짐
- 같은 IATA의 중복 membership
- child count를 읽은 직후 다른 transaction이 append 시도
- hash가 같지만 seal version이 unknown
- legacy migration seal과 canonical seal의 구분

### 실패 조건

- `VALIDATED` 상태만 보고 child 집합을 trust함
- DB의 비결정적 row 순서로 closure hash를 만듦
- seal 이후 append를 애플리케이션 코드에서만 막음
- publish 시 seal 존재 여부만 보고 lineage를 재검증하지 않음
- 불완전 후보를 “현재 있는 부분만” 봉인함

### 필요한 제약사항

- canonical payload는 business field만 포함하고 DB 생성 순서나 object repr에 의존하지 않는다.
- candidate별 lock key는 seal과 모든 child membership write에서 동일해야 한다.
- seal은 append-only/immutable이며 새 semantics는 새 version으로 구분한다.

## 구현 후 자가 검증

- [ ] 같은 aggregate는 row 조회 순서가 달라도 같은 closure hash인가?
- [ ] schedule/duration 집합의 누락·중복·extra가 모두 거부되는가?
- [ ] seal 후 child insert/update/delete가 DB 수준에서 막히는가?
- [ ] seal transaction과 동시 child append 중 하나만 선행하고 다른 하나가 일관된 상태를 보는가?
- [ ] 저장 count와 현재 count가 하나라도 다르면 게시 검증이 실패하는가?
- [ ] typed child 값 변조를 재계산 hash가 감지하는가?
- [ ] unknown seal version을 조용히 현재 version으로 해석하지 않는가?
- [ ] source/parse/duration lineage까지 게시 전에 다시 확인하는가?

## 구현 후 설명할 것

- row immutability와 aggregate closure가 다른 invariant인 이유
- 결정적 정렬 기준을 IATA/typed hash에 둔 이유
- count와 hash를 함께 보관한 방어 심층화
- seal/append race를 같은 advisory lock으로 닫는 방식
- stage-time validation과 publish-time revalidation의 책임 차이

## 원본 확인 위치

- 파일: `travel_windows/publication.py`
- 함수: `_schedule_payload_from_models`, `_duration_payload_from_model`, `_closure_fingerprint`, `stage_flight_evidence`, `_load_valid_candidate`
- 모델: `travel_windows/models.py::FlightCandidateSeal`, `FlightCandidateDuration`, `FlightScheduleRevision`
- Commit: `254c0a4 fix(publication): seal typed flight candidates`
- 후속 Commit: `1b2a0df fix(db): close flight rollback and seal races`
- DB guard: `travel_windows/migrations/0008_seal_flight_candidates.py::travel_windows_guard_candidate_seal_insert`, `travel_windows_guard_sealed_schedule_insert`, `travel_windows_guard_candidate_duration_insert`
- 관련 테스트: `travel_windows/tests/test_publication_sealing.py`
- 연관 lifecycle 테스트: `travel_windows/tests/test_review_lifecycle.py::test_worker_stages_typed_candidate_without_publication_or_pointer_move`
