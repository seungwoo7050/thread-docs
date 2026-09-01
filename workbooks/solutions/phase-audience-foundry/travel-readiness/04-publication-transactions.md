# 게시 트랜잭션·DB closure·롤백 워크북

이 문서는 검증된 후보가 운영자 승인과 감사 이력을 거쳐 current pointer가 되는 과정, ORM을 우회해도 유지되어야 하는 multi-row invariant, 과거 상태를 복구하되 이력을 덮어쓰지 않는 rollback, 실패 자체를 안전하게 감사하는 경계를 다룬다.

<a id="w10"></a>

---

# [travel_windows/publication.py::approve_flight_candidate, _advance_pointer] W10 — 운영자 승인·감사·포인터 이동의 단일 원자 작업 (S)

## 면접 질문

`approve_flight_candidate`는 한 durable transaction 안에서 registry advisory lock, current pointer row lock, expected version 검사, candidate 재검증, actor 권한 확인, review/publication/duration snapshot/audit 생성, pointer CAS를 순서대로 수행합니다. 이 중 일부만 transaction 밖으로 빼면 어떤 partial state가 생기는지 설명해 보세요.

꼬리 질문:

1. pointer row를 `select_for_update`로 잠갔는데도 `_advance_pointer`가 `WHERE version = expected` 조건부 update를 하는 이유는 무엇인가요?
2. actor를 ID만 믿지 않고 DB에서 다시 row lock하고 active/staff/permission을 확인하는 이유는 무엇인가요?
3. publication에 live duration revision 링크를 직접 조회하게 하지 않고 `FlightPublicationDuration` snapshot membership을 별도로 남기는 이유는 무엇인가요?
4. audit insert가 실패하면 review와 publication도 rollback되어야 하는 이유는 무엇인가요?

## 30초 모범 답변

승인은 review·publication·audit·current pointer가 함께 닫혀야 의미가 있습니다. pointer를 잠가 동일 generation의 writer를 직렬화하고, caller의 expected version으로 stale 의도를 조기에 거부하며, 마지막 conditional update로 CAS invariant를 다시 확인합니다. actor도 transaction 안에서 다시 잠가 권한 변경 race를 막습니다. 어느 insert나 audit가 실패해도 전체가 rollback되어 current pointer만 움직이거나 orphan review가 남지 않습니다.

## 답변 핵심 키워드

- atomic publication closure
- row lock + CAS
- stale intent
- authorization recheck
- immutable snapshot membership
- all-or-nothing audit

## 꼬리 질문과 핵심 답변

### pessimistic row lock과 optimistic version을 함께 쓰는 것이 중복 아닌가요?

row lock은 실행 중 동시 writer를 직렬화하고, expected version은 사용자가 본 상태가 아직 current인지 표현합니다. lock을 늦게 얻은 writer도 자신의 오래된 의도를 자동으로 새 current에 적용하지 않고 `STALE_POINTER`로 실패합니다. 조건부 update는 코드·trigger 변경에도 마지막 CAS 방어가 됩니다.

### registry advisory lock은 pointer row lock과 무엇이 다른가요?

pointer가 아직 생성되지 않았거나 여러 관련 table의 write 순서를 맞춰야 할 때 공통 논리 scope를 먼저 직렬화합니다. 이후 실제 singleton pointer row lock은 row-level 현재 상태를 보호합니다. 모든 경로가 같은 lock order를 따라야 deadlock을 줄일 수 있습니다.

### candidate 검증을 transaction 안에서 다시 하는 비용은 괜찮나요?

여러 child와 lineage를 lock·검증해 비용은 늘지만, 검토한 evidence와 게시할 evidence가 같은 시점의 집합이어야 합니다. candidate seal과 인덱스로 범위를 제한하고, 게시 빈도가 검색보다 훨씬 낮다는 도메인 특성상 정합성을 우선합니다.

### publication row를 만든 뒤 pointer만 실패하면 publication history를 남겨도 되지 않나요?

이 모델에서 publication은 성공 audit와 current pointer 이동으로 닫힌 operator action입니다. pointer에 도달하지 않은 publication을 “게시 이력”으로 남기면 generation·supersedes chain과 review 의미가 모호해집니다. 별도 draft/event 모델이 필요하다면 다른 상태 기계로 설계해야 합니다.

### unauthorized 시도도 같은 transaction에 audit하면 되지 않나요?

실패 transaction이 rollback되면 audit도 사라집니다. 성공 closure audit는 주 transaction 안에, 실패 audit는 rollback 뒤 별도 durable transaction에 기록해야 합니다. 이 차이는 W13에서 다룹니다.

## 백지 구현

### 구현 목표

검증된 candidate를 운영자 승인으로 current pointer에 게시하는 원자적 함수 구현. 동시에 두 publisher가 같은 expected version을 사용하면 정확히 하나만 성공해야 한다.

### 인터페이스

`publish_candidate(candidate_id: UUID, actor_id: int, expected_version: int) -> PublishOutcome`

`PublishOutcome`은 `PUBLISHED`, `STALE_POINTER`, `NOT_AUTHORIZED`, `INVALID_EVIDENCE`, `TRANSACTION_ABORTED` 중 하나와 성공 시 publication ID/generation/new pointer version을 가진다.

```python
import hashlib
import uuid


class PublishRejected(Exception):
    def __init__(self, code):
        self.code = code


def _audit_identity(review_id, publication_id, prior_id, correlation_id):
    canonical = "\n".join(map(str, (
        "FLIGHT_PUBLISH_V1", review_id, publication_id,
        prior_id or "", correlation_id,
    ))) + "\n"
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


def publish_candidate(candidate_id, actor_id, expected_version):
    if type(expected_version) is not int or expected_version < 0:
        return PublishOutcome("STALE_POINTER")
    try:
        with repository.begin_durable():
            # publish/reject/rollback 모두 이 순서를 공유한다.
            repository.lock_publication_registry()
            pointer = repository.lock_current_pointer("FLIGHT")
            if pointer.version != expected_version:
                raise PublishRejected("STALE_POINTER")

            candidate = repository.load_and_revalidate_sealed_candidate(
                candidate_id
            )
            if candidate is None or repository.has_review_history(candidate_id):
                raise PublishRejected("INVALID_EVIDENCE")

            actor = repository.lock_actor(actor_id)
            if (actor is None or not actor.is_active or not actor.is_staff
                    or not actor.has_permission("publish_flight")):
                raise PublishRejected("NOT_AUTHORIZED")

            generation = pointer.version + 1
            prior_id = pointer.current_publication_id
            correlation_id = uuid.uuid4()
            review = repository.create_review(
                module="FLIGHT",
                subject_id=candidate.id,
                decision="APPROVED",
                reason="PUBLISH",
                actor_id=actor.id,
                correlation_id=correlation_id,
            )
            publication = repository.create_publication(
                review_id=review.id,
                subject_id=candidate.id,
                generation=generation,
                supersedes_id=prior_id,
                rollback_target_id=None,
                source_snapshot=candidate.source_snapshot,
            )
            copied = repository.create_publication_children(
                publication.id, candidate.duration_memberships
            )
            if copied != len(candidate.duration_memberships):
                raise PublishRejected("INVALID_EVIDENCE")

            repository.create_success_audit(
                action="REVIEW_PUBLISH",
                review_id=review.id,
                publication_id=publication.id,
                prior_publication_id=prior_id,
                rollback_target_id=None,
                actor_id=actor.id,
                correlation_id=correlation_id,
                identity_sha256=_audit_identity(
                    review.id, publication.id, prior_id, correlation_id
                ),
            )
            changed = repository.cas_pointer(
                pointer_id=pointer.id,
                expected_version=expected_version,
                expected_current_id=prior_id,
                new_publication_id=publication.id,
                new_version=generation,
            )
            if changed != 1:
                raise PublishRejected("STALE_POINTER")

        return PublishOutcome(
            "PUBLISHED",
            publication_id=publication.id,
            generation=generation,
            pointer_version=generation,
        )
    except PublishRejected as rejected:
        return PublishOutcome(rejected.code)
    except Exception:
        return PublishOutcome("TRANSACTION_ABORTED")
```

### 입력

- 아직 review/publication 이력이 없는 sealed candidate ID
- 요청 actor ID
- caller가 읽은 non-negative pointer version

### 출력

- 성공: 새 publication과 generation 식별자
- 실패: 폐쇄형 code만 반환하고 partial success ID를 노출하지 않는다.

### 반드시 만족해야 할 조건

- 하나의 DB transaction에서 논리 registry lock과 pointer row lock을 같은 순서로 획득한다.
- pointer version이 expected와 다르면 어떤 review/publication/audit도 만들지 않는다.
- candidate seal, child closure, source/parse lineage를 transaction 안에서 재검증한다.
- actor row를 lock하고 active/staff/permission을 현재 값으로 확인한다.
- review, publication, publication child snapshot, success audit, pointer update를 모두 같은 transaction에 둔다.
- generation은 `old pointer version + 1`, supersedes는 old current publication이다.
- pointer update는 ID와 expected version 조건을 가진 CAS이며 정확히 한 행을 갱신해야 한다.

### 경계 조건

- 초기 pointer version 0/current null
- 같은 expected version을 가진 두 concurrent publisher
- lock 대기 중 actor 권한이 철회됨
- candidate child가 seal과 맞지 않음
- review insert 성공 후 publication insert 실패
- audit insert 실패
- CAS update row count가 0 또는 예상 밖 값

### 실패 조건

- stale writer가 최신 pointer를 자동으로 이어 게시함
- unauthorized actor로 review row가 남음
- audit 실패에도 pointer가 이동함
- publication child가 candidate 전체 집합과 다름
- 예외 문자열·SQL 상세를 outcome에 포함함

### 필요한 제약사항

- transaction retry를 한다면 전체 함수의 명시적 상위 정책으로 제한하고, 내부에서 operator action을 중복 생성하지 않는다.
- lock order를 모든 publish/reject/rollback/DB trigger 경로에서 일관되게 유지한다.
- 성공 audit identity는 review/publication/prior/correlation을 결정적으로 결합한다.

## 구현 후 자가 검증

- [ ] 초기 게시와 후속 게시 generation/supersedes가 연속인가?
- [ ] 같은 expected version의 동시 publisher 중 하나만 성공하는가?
- [ ] stale·unauthorized·invalid evidence에 partial row가 전혀 남지 않는가?
- [ ] actor 권한을 transaction 안의 현재 row로 검증하는가?
- [ ] publication child snapshot이 candidate membership과 정확히 같은가?
- [ ] audit insert 실패가 review/publication/pointer를 모두 rollback하는가?
- [ ] pointer CAS가 정확히 한 행 갱신을 요구하는가?
- [ ] 성공 outcome의 generation과 pointer version이 DB 상태와 일치하는가?

## 구현 후 설명할 것

- row lock과 expected-version CAS가 각각 막는 race
  - 모범답변: row lock은 같은 pointer를 읽고 쓰는 transaction을 직렬화해 중간 상태 관찰을 막습니다. expected-version CAS는 대기 후 깨어난 stale writer도 caller가 본 버전과 다르면 갱신하지 못하게 하는 최종 조건입니다.
- review/publication/audit/pointer를 하나의 closure로 본 이유
  - 모범답변: review만 있거나 audit 없이 pointer만 움직이면 현재 데이터의 승인 근거를 재구성할 수 없습니다. 네 요소와 publication child snapshot이 함께 commit되거나 모두 rollback돼야 게시 상태가 의미를 가집니다.
- actor 권한을 transaction 안에서 다시 확인한 이유
  - 모범답변: 요청 인증 뒤 lock을 기다리는 동안 계정 비활성화나 권한 철회가 일어날 수 있습니다. actor row를 잠그고 현재 active/staff/permission을 확인해야 오래된 인증 판단으로 게시하지 않습니다.
- 게시 시점에 candidate 전체 lineage를 재검증하는 비용과 이점
  - 모범답변: child와 source·parse lineage를 다시 읽고 hash하는 비용이 들지만 검토 이후 direct SQL, 버그, race로 aggregate가 바뀐 경우를 차단합니다. current pointer는 제품 신뢰 경계이므로 stage 결과만 신뢰하지 않습니다.
- publication child snapshot이 과거 재현성을 제공하는 방식
  - 모범답변: publication에 당시 candidate의 exact duration membership을 복사하면 이후 새 candidate나 자료가 생겨도 그 generation이 어떤 공항별 증거를 공개했는지 재현할 수 있습니다. rollback도 이 snapshot을 그대로 복제합니다.

## 원본 확인 위치

- flight 구현: `travel_windows/publication.py::approve_flight_candidate`, `_locked_pointer`, `_advance_pointer`, `_load_valid_candidate`, `_create_audit`
- 공통 entry/warning 구현: `reviews/publication.py::_publish_candidate_inner`, `_lock_module`, `_lock_authorized_actor`
- Commit: `0a3f04a feat(publication): require operator review for flights`
- Commit: `8f85a69 feat(reviews): publish approved facts atomically`
- 관련 테스트: `travel_windows/tests/test_review_lifecycle.py::test_authorized_approval_commits_review_publication_audit_and_pointer`
- 실패 원자성 테스트: 같은 파일의 stale/unauthorized/audit failure 테스트
- 공통 테스트: `reviews/tests/test_publication.py`

<a id="w11"></a>

---

# [reviews/migrations/0001_initial.py::reviews_enforce_deferred_closure] W11 — 애플리케이션 우회를 막는 deferred DB closure (A)

## 면접 질문

review, publication, audit, pointer는 한 transaction에서 순서대로 insert/update되므로 각 statement 직후에는 일시적으로 불완전합니다. `DEFERRABLE INITIALLY DEFERRED` constraint trigger가 왜 필요한지, 일반 `CHECK`, foreign key, 즉시 trigger만으로는 어떤 invariant를 표현하기 어려운지 설명해 보세요.

꼬리 질문:

1. approved review와 rejected review는 commit 시 각각 어떤 closure shape를 가져야 하나요?
2. closure trigger가 모든 관련 table에 붙어 있는 이유는 무엇인가요?
3. 애플리케이션에서 transaction을 잘 작성했는데 DB trigger까지 두는 비용을 정당화할 수 있나요?
4. trigger가 advisory lock과 pointer row를 일정한 순서로 잡지 않으면 어떤 문제가 생기나요?

## 30초 모범 답변

이 invariant는 한 행이 아니라 review·publication·audit·pointer 전체의 관계입니다. 정상 transaction도 중간 statement에서는 closure가 완성되지 않으므로 즉시 검사는 오탐을 냅니다. deferred constraint trigger가 commit 직전에 최종 상태를 보고, 승인에는 publication 1개·성공 audit 1개·current pointer exact match를, 거절에는 publication 0개·거절 audit 1개를 요구합니다. ORM 밖의 SQL도 같은 규칙을 지키게 하고, prelock trigger와 공통 lock order로 동시 우회와 deadlock 위험을 줄입니다.

## 답변 핵심 키워드

- multi-row invariant
- deferred constraint trigger
- commit-time closure
- ORM bypass defense
- exact cardinality
- lock ordering

## 꼬리 질문과 핵심 답변

### 일반 CHECK constraint가 왜 부족한가요?

CHECK는 기본적으로 현재 행 값에 적합하고 다른 table의 동적 count·current pointer·최신 review 관계를 안전하게 표현하기 어렵습니다. PostgreSQL은 subquery가 있는 일반 CHECK도 이 용도에 맞지 않습니다.

### foreign key와 unique constraint가 제공하는 것은 무엇이고 빠지는 것은 무엇인가요?

참조 존재와 1:1 cardinality 일부는 보장하지만, “APPROVED면 정확히 publication 1개와 success audit 1개”, “pointer가 바로 그 publication”, “generation이 predecessor+1” 같은 조건부 closure 전체는 보장하지 못합니다.

### 관련 table 모두에 trigger를 붙이면 같은 closure를 여러 번 검사하지 않나요?

그럴 수 있습니다. 하지만 어느 table만 직접 변경해도 commit 검사가 예약되어야 합니다. 정확성 우선의 중복이며, transaction 규모가 작고 게시 빈도가 낮아 허용됩니다. 필요하면 transition table이나 명시적 validation function 호출로 최적화할 수 있지만 우회 가능성을 다시 평가해야 합니다.

### deferred trigger가 실패하면 이미 수행한 statement는 어떻게 되나요?

commit이 실패해 transaction 전체가 rollback됩니다. 애플리케이션은 이를 폐쇄형 `TRANSACTION_ABORTED` 등으로 변환하고, 실패 audit가 필요하면 rollback 이후 새 transaction에서 기록합니다.

### warning snapshot child count closure도 같은 패턴인가요?

네. root row를 먼저 만들고 0..N fact child를 이어 insert하므로 즉시 root trigger는 아직 불완전한 상태를 봅니다. deferred trigger가 commit 시 count, 1..N position, typed fingerprint를 재계산합니다.

## 백지 구현

### 구현 목표

다음 네 table의 게시 closure를 commit 시 검증하는 PostgreSQL 설계를 작성한다: `review`, `publication`, `audit`, `current_pointer`. 전체 migration을 작성할 필요는 없고 trigger function의 검사 항목과 trigger 배치만 구현한다.

### 인터페이스

SQL 함수 개념 인터페이스:

`enforce_publication_closure() RETURNS trigger`

관련 table의 `AFTER INSERT` 또는 pointer의 `AFTER UPDATE`에 `DEFERRABLE INITIALLY DEFERRED` constraint trigger로 연결한다.

```sql
CREATE OR REPLACE FUNCTION enforce_publication_closure()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    review_id_to_check uuid;
    r review%ROWTYPE;
    p publication%ROWTYPE;
    a audit%ROWTYPE;
    ptr current_pointer%ROWTYPE;
    publication_count integer;
    audit_count integer;
    predecessor_count integer;
BEGIN
    -- 별도 failure audit는 review closure의 구성원이 아니며, 그 shape는
    -- 행 제약/전용 guard가 검사한다.
    IF TG_TABLE_NAME = 'audit' AND NEW.outcome <> 'SUCCEEDED' THEN
        IF NEW.review_id IS NOT NULL OR NEW.publication_id IS NOT NULL THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
        RETURN NULL;
    END IF;

    IF TG_TABLE_NAME = 'review' THEN
        review_id_to_check := NEW.id;
    ELSIF TG_TABLE_NAME = 'publication' THEN
        review_id_to_check := NEW.review_id;
    ELSIF TG_TABLE_NAME = 'audit' THEN
        review_id_to_check := NEW.review_id;
    ELSIF TG_TABLE_NAME = 'current_pointer' THEN
        SELECT review_id INTO review_id_to_check
          FROM publication WHERE id = NEW.current_publication_id;
    END IF;

    SELECT * INTO STRICT r FROM review WHERE id = review_id_to_check;
    SELECT count(*) INTO publication_count
      FROM publication WHERE review_id = r.id;
    SELECT count(*) INTO audit_count
      FROM audit WHERE review_id = r.id;

    IF r.decision = 'REJECTED' THEN
        IF publication_count <> 0 OR audit_count <> 1 THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
        SELECT * INTO STRICT a FROM audit WHERE review_id = r.id;
        IF a.action <> 'REVIEW_REJECT' OR a.outcome <> 'SUCCEEDED'
           OR a.publication_id IS NOT NULL
           OR a.actor_id IS DISTINCT FROM r.actor_id
           OR a.module IS DISTINCT FROM r.module
           OR a.subject_id IS DISTINCT FROM r.subject_id
           OR a.correlation_id IS DISTINCT FROM r.correlation_id
           OR EXISTS (
               SELECT 1 FROM current_pointer cp
               JOIN publication published ON published.id = cp.current_publication_id
               WHERE published.review_id = r.id
           ) THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
        RETURN NULL;
    END IF;

    IF r.decision <> 'APPROVED'
       OR publication_count <> 1 OR audit_count <> 1 THEN
        RAISE EXCEPTION USING ERRCODE = '23514',
            MESSAGE = 'publication closure violated';
    END IF;

    SELECT * INTO STRICT p FROM publication WHERE review_id = r.id;
    SELECT * INTO STRICT a FROM audit WHERE review_id = r.id;
    SELECT * INTO STRICT ptr FROM current_pointer
      WHERE module = r.module AND subject_id = r.subject_id;

    IF p.subject_id IS DISTINCT FROM r.subject_id
       OR ptr.current_publication_id IS DISTINCT FROM p.id
       OR ptr.version IS DISTINCT FROM p.generation
       OR a.publication_id IS DISTINCT FROM p.id
       OR a.prior_publication_id IS DISTINCT FROM p.supersedes_id
       OR a.actor_id IS DISTINCT FROM r.actor_id
       OR a.module IS DISTINCT FROM r.module
       OR a.subject_id IS DISTINCT FROM r.subject_id
       OR a.correlation_id IS DISTINCT FROM r.correlation_id
       OR a.outcome <> 'SUCCEEDED' THEN
        RAISE EXCEPTION USING ERRCODE = '23514',
            MESSAGE = 'publication closure violated';
    END IF;

    IF p.generation = 1 THEN
        IF p.supersedes_id IS NOT NULL THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
    ELSE
        SELECT count(*) INTO predecessor_count
          FROM publication previous
          JOIN review previous_review ON previous_review.id = previous.review_id
         WHERE previous.id = p.supersedes_id
           AND previous.generation = p.generation - 1
           AND previous_review.module = r.module
           AND previous_review.subject_id = r.subject_id;
        IF predecessor_count <> 1 THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
    END IF;

    IF p.rollback_target_id IS NULL THEN
        IF r.reason <> 'PUBLISH' OR a.action <> 'REVIEW_PUBLISH'
           OR a.rollback_target_id IS NOT NULL THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
    ELSE
        IF r.reason <> 'ROLLBACK' OR a.action <> 'REVIEW_ROLLBACK'
           OR a.rollback_target_id IS DISTINCT FROM p.rollback_target_id THEN
            RAISE EXCEPTION USING ERRCODE = '23514',
                MESSAGE = 'publication closure violated';
        END IF;
    END IF;
    RETURN NULL;
EXCEPTION
    WHEN NO_DATA_FOUND OR TOO_MANY_ROWS THEN
        RAISE EXCEPTION USING ERRCODE = '23514',
            MESSAGE = 'publication closure violated';
END;
$$;

CREATE CONSTRAINT TRIGGER review_deferred_closure
AFTER INSERT ON review DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW EXECUTE FUNCTION enforce_publication_closure();

CREATE CONSTRAINT TRIGGER publication_deferred_closure
AFTER INSERT ON publication DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW EXECUTE FUNCTION enforce_publication_closure();

CREATE CONSTRAINT TRIGGER audit_deferred_closure
AFTER INSERT ON audit DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW EXECUTE FUNCTION enforce_publication_closure();

CREATE CONSTRAINT TRIGGER pointer_deferred_closure
AFTER INSERT OR UPDATE OF current_publication_id, version ON current_pointer
DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW EXECUTE FUNCTION enforce_publication_closure();
```

### 입력

- review: module, decision, actor, subject, correlation, reason
- publication: review ID, generation, supersedes, 선택적 rollback target, subject
- audit: review/publication/prior/rollback target, action, actor, outcome, correlation
- pointer: current publication ID, version

### 출력

- 정상 closure면 commit 허용
- 불일치면 고정된 check violation으로 transaction 전체 거부

### 반드시 만족해야 할 조건

- `REJECTED` review는 publication 0개, 성공 `REVIEW_REJECT` audit 정확히 1개이며 pointer target이 되면 안 된다.
- `APPROVED` review는 publication 정확히 1개, 성공 audit 정확히 1개여야 한다.
- pointer는 해당 publication을 current로 가리키고 version은 publication generation과 일치해야 한다.
- audit actor/module/subject/correlation은 review와 정확히 같아야 한다.
- publication generation 1은 predecessor가 없고, 이후 generation은 같은 scope의 `generation - 1` predecessor를 가져야 한다.
- publish와 rollback reason/action/rollback-target shape가 서로 일치해야 한다.
- direct SQL로 어느 관련 table만 변경해도 closure 검사가 예약되어야 한다.

### 경계 조건

- 첫 publication과 후속 publication
- review만 insert하고 commit
- review/publication은 있으나 audit가 없음
- audit가 두 개이거나 실패 audit만 있음
- pointer가 다른 publication을 가리킴
- actor나 correlation 하나만 불일치
- 중간 statement에서는 불완전하지만 commit 전에는 완성되는 정상 transaction

### 실패 조건

- 즉시 trigger 때문에 정상 multi-statement transaction이 첫 insert에서 실패함
- 한 table에만 trigger가 있어 다른 table direct update가 검사를 피함
- count만 맞고 identity가 다른 row를 허용함
- trigger 내부 lock 순서가 애플리케이션과 달라 deadlock cycle을 만듦
- 예외 후 일부 row가 commit됨

### 필요한 제약사항

- PostgreSQL constraint trigger를 사용한다.
- 함수는 raw 민감값을 예외 메시지에 포함하지 않는다.
- 게시 scope별 공통 advisory lock 또는 pointer lock order를 문서화한다.
- trigger 비용을 bounded하게 유지하도록 singleton pointer와 indexed foreign key를 사용한다.

## 구현 후 자가 검증

- [ ] 정상 transaction은 중간 불완전 상태를 허용하고 commit 시 성공하는가?
- [ ] review/publication/audit/pointer 중 하나라도 빠지거나 extra이면 commit이 실패하는가?
- [ ] approved/rejected closure가 서로 다른 exact cardinality를 요구하는가?
- [ ] actor·subject·correlation·prior·rollback target identity를 모두 비교하는가?
- [ ] generation과 predecessor scope/연속성을 검사하는가?
- [ ] 각 관련 table의 direct SQL 우회를 테스트했는가?
- [ ] 동시 게시·롤백에서 lock order가 일관적인가?
- [ ] trigger 실패가 transaction 전체 rollback을 일으키는가?

## 구현 후 설명할 것

- deferred 검사가 필요한 정상 중간 상태
  - 모범답변: 정상 publish도 review, publication, audit, pointer를 여러 statement로 만드므로 첫 insert 직후에는 closure가 불완전합니다. initially deferred trigger는 transaction 내부 중간 상태를 허용하고 commit 직전 완성된 집합만 검사합니다.
- 행 단위 제약과 aggregate closure 제약의 차이
  - 모범답변: `generation > 0` 같은 행 제약은 한 row만 보면 판단할 수 있습니다. publication 개수, success audit identity, predecessor와 current pointer의 일치는 여러 table을 함께 봐야 하는 aggregate invariant입니다.
- 애플리케이션 검증과 DB 검증을 함께 둔 defense-in-depth
  - 모범답변: 애플리케이션은 좋은 failure code와 lock 흐름을 제공하지만 direct SQL, 다른 worker, 코드 버그가 우회할 수 있습니다. DB closure는 어떤 쓰기 경로에서도 불완전한 게시 상태의 commit 자체를 거부합니다.
- 관련 table 모두에 constraint trigger를 둔 이유
  - 모범답변: review에만 trigger가 있으면 나중의 audit 또는 pointer direct write가 검사를 예약하지 않을 수 있습니다. closure를 바꿀 수 있는 각 table에서 deferred 검사를 예약해야 우회 경로가 없습니다.
- lock ordering과 trigger 성능 trade-off
  - 모범답변: trigger도 애플리케이션과 같은 registry·pointer 순서를 지켜야 deadlock cycle을 피합니다. commit마다 join/count 비용이 있지만 singleton pointer와 indexed FK, 한 review 범위 조회로 비용을 제한합니다.

## 원본 확인 위치

- 공통 entry/warning DB closure: `reviews/migrations/0001_initial.py::reviews_enforce_deferred_closure`
- country-scoped 후속 진화: `reviews/migrations/0002_country_scoped_publications.py::_rewrite_functions`가 같은 trigger function을 국가·여권정책 포인터 범위로 교체
- 연결 trigger: 같은 파일의 `reviews_review_deferred_closure`, `reviews_publication_deferred_closure`, `reviews_audit_deferred_closure`, pointer deferred closure들
- flight pointer guard: `travel_windows/migrations/0007_flight_review_lifecycle.py::travel_windows_guard_flight_pointer`
- candidate seal guard: `travel_windows/migrations/0008_seal_flight_candidates.py`
- warning root/child closure: `travel_warnings/migrations/0004_warning_snapshot_facts.py::travel_warnings_validate_snapshot_closure`
- Commit: `7437bfe fix(db): enforce reviewed flight pointer closure`
- Commit: `1b2a0df fix(db): close flight rollback and seal races`
- 연관 Commit: `dd2d420 feat(source): publish supported-country travel evidence`
- 관련 테스트: `travel_windows/tests/test_review_lifecycle.py`의 database bypass, causal time, pointer closure 테스트
- 공통 테스트: `reviews/tests/test_publication.py`의 deferred closure와 concurrent publisher 테스트

<a id="w12"></a>

---

# [travel_windows/publication.py::_target_is_ancestor, rollback_flight_publication] W12 — append-only strict-ancestor rollback (S)

## 면접 질문

`rollback_flight_publication`은 current pointer를 과거 publication으로 되돌려 꽂거나 기존 row를 수정하지 않고, 과거 evidence를 복제한 새 generation을 current 뒤에 append합니다. 이 방식이 일반적인 “pointer rewind”보다 감사·동시성·향후 게시에 유리한 이유를 설명해 보세요.

꼬리 질문:

1. rollback target을 arbitrary publication이 아니라 current의 strict ancestor로 제한하는 이유는 무엇인가요?
2. `_target_is_ancestor`에 visited set이 필요한 이유는 무엇인가요?
3. target schedule revision만 같으면 충분하지 않고 publication duration membership까지 exact set 비교하는 이유는 무엇인가요?
4. 새 rollback publication의 `supersedes`와 `rollback_target`은 각각 무엇을 가리켜야 하나요?

## 30초 모범 답변

rollback도 하나의 현재 시점 operator action이므로 과거를 지우거나 pointer를 역행시키지 않고 새 generation으로 기록해야 합니다. 새 row는 current를 `supersedes`해 history를 연속시키고, 복구하려는 strict ancestor를 `rollback_target`으로 별도 표시합니다. target의 schedule·source snapshot·duration membership을 exact copy하고, ancestor 탐색은 lock과 cycle 방어를 사용합니다. 그래서 누가 언제 무엇으로 복구했는지 audit가 남고 다음 게시도 단조 증가 generation에서 계속됩니다.

## 답변 핵심 키워드

- append-only rollback
- strict ancestor
- monotonic generation
- supersedes vs rollback target
- exact evidence restoration
- cycle defense

## 꼬리 질문과 핵심 답변

### pointer를 과거 row로 직접 바꾸면 어떤 이력이 사라지나요?

rollback action 자체를 나타내는 generation, actor review, audit, 당시 current와의 predecessor 관계가 사라집니다. 이후 다시 게시할 때 generation을 어떻게 이어야 하는지도 모호해집니다.

### 다른 branch의 publication을 target으로 허용하면 왜 위험한가요?

현재 history에서 실제로 게시된 과거 상태가 아닐 수 있고 scope·source 계약이 다를 수 있습니다. arbitrary 복사나 branch merge는 rollback이 아니라 별도 republish/review 작업으로 다뤄야 합니다.

### target과 current가 같으면 왜 거부하나요?

상태 변화가 없는 rollback generation을 무한히 만들 수 있고 operator 의도 오류를 숨깁니다. strict ancestor만 허용해 실제 과거 복구라는 의미를 유지합니다.

### ancestor traversal에서 row lock을 거는 이유는 무엇인가요?

publication이 append-only라면 predecessor 값은 변하지 않아야 하지만, DB guard 우회나 concurrent 관련 write까지 같은 transaction 관점에서 고정하고 target chain의 존재를 확실히 하기 위한 방어층입니다.

### target evidence를 현재 candidate validator로 다시 확인하면 과거 version 호환성 문제가 생기지 않나요?

실제로 이 저장소는 legacy seal/version을 명시적으로 구분해 허용 범위를 제한합니다. 과거를 무조건 trust하지 않고, 지원하는 legacy 계약만 명시적으로 검증하는 것이 안전합니다.

## 백지 구현

### 구현 목표

단일 연결 리스트 형태의 append-only publication history에서 current의 과거 ancestor를 복구하는 새 generation을 만든다. 기존 publication과 pointer version을 감소·수정하지 않는다.

### 인터페이스

`rollback(target_publication_id: UUID, actor_id: int, expected_version: int) -> RollbackOutcome`

도우미:

`is_ancestor(current: Publication, target_id: UUID, load_and_lock: Callable) -> bool`

```python
import uuid


class RollbackRejected(Exception):
    def __init__(self, code):
        self.code = code


def is_ancestor(current, target_id, load_and_lock):
    next_id = current.supersedes_id  # current 자신은 strict ancestor가 아니다.
    visited = {current.id}
    while next_id is not None:
        if next_id in visited:
            return False
        visited.add(next_id)
        publication = load_and_lock(next_id)
        if publication is None:
            return False
        if publication.id == target_id:
            return True
        next_id = publication.supersedes_id
    return False


def rollback(target_publication_id, actor_id, expected_version):
    if type(expected_version) is not int or expected_version <= 0:
        return RollbackOutcome("STALE_POINTER")
    try:
        with repository.begin_durable():
            repository.lock_publication_registry()
            pointer = repository.lock_current_pointer("FLIGHT")
            if (pointer.version != expected_version
                    or pointer.current_publication_id is None):
                raise RollbackRejected("STALE_POINTER")

            current = repository.lock_publication(pointer.current_publication_id)
            if (current is None or current.id == target_publication_id
                    or not is_ancestor(
                        current,
                        target_publication_id,
                        repository.lock_publication,
                    )):
                raise RollbackRejected("INVALID_TARGET")

            target = repository.lock_publication(target_publication_id)
            target_candidate = repository.load_and_revalidate_sealed_candidate(
                target.subject_id
            )
            target_children = repository.lock_publication_children(target.id)
            if (target_candidate is None
                    or repository.membership_identity(target_children)
                    != repository.membership_identity(
                        target_candidate.duration_memberships
                    )):
                raise RollbackRejected("INVALID_EVIDENCE")

            actor = repository.lock_actor(actor_id)
            if (actor is None or not actor.is_active or not actor.is_staff
                    or not actor.has_permission("rollback_flight")):
                raise RollbackRejected("NOT_AUTHORIZED")

            generation = pointer.version + 1
            correlation_id = uuid.uuid4()
            review = repository.create_review(
                module="FLIGHT",
                subject_id=target.subject_id,
                decision="APPROVED",
                reason="ROLLBACK",
                actor_id=actor.id,
                correlation_id=correlation_id,
            )
            restored = repository.create_publication(
                review_id=review.id,
                subject_id=target.subject_id,
                generation=generation,
                supersedes_id=current.id,
                rollback_target_id=target.id,
                source_snapshot=target.source_snapshot,
            )
            if repository.copy_publication_children(
                    from_publication_id=target.id,
                    to_publication_id=restored.id) != len(target_children):
                raise RollbackRejected("INVALID_EVIDENCE")
            repository.create_success_audit(
                action="REVIEW_ROLLBACK",
                review_id=review.id,
                publication_id=restored.id,
                prior_publication_id=current.id,
                rollback_target_id=target.id,
                actor_id=actor.id,
                correlation_id=correlation_id,
            )
            changed = repository.cas_pointer(
                pointer_id=pointer.id,
                expected_version=expected_version,
                expected_current_id=current.id,
                new_publication_id=restored.id,
                new_version=generation,
            )
            if changed != 1:
                raise RollbackRejected("STALE_POINTER")

        return RollbackOutcome(
            "ROLLED_BACK",
            publication_id=restored.id,
            generation=generation,
            pointer_version=generation,
        )
    except RollbackRejected as rejected:
        return RollbackOutcome(rejected.code)
    except Exception:
        return RollbackOutcome("TRANSACTION_ABORTED")
```

### 입력

- target publication ID
- rollback 권한을 검증할 actor ID
- caller가 읽은 양의 current pointer version

### 출력

- 성공: `ROLLED_BACK`, 새 publication ID, 증가한 generation/version
- 실패: `STALE_POINTER`, `INVALID_TARGET`, `INVALID_EVIDENCE`, `NOT_AUTHORIZED`, `TRANSACTION_ABORTED`

### 반드시 만족해야 할 조건

- registry/pointer를 W10과 같은 lock order로 잠근다.
- expected version과 current version이 같아야 한다.
- target은 current 자신이 아니며 current의 predecessor chain에 있어야 한다.
- ancestor traversal은 missing predecessor와 cycle을 안전하게 실패 처리한다.
- target candidate closure와 target publication child snapshot이 exact set으로 같아야 한다.
- 새 publication generation은 current version + 1이다.
- 새 publication은 current를 supersedes하고 target을 rollback target으로 기록한다.
- source snapshot과 child membership은 target에서 복원한다.
- review, publication, child snapshot, audit, pointer CAS를 한 transaction에 둔다.

### 경계 조건

- history 길이 1에서 rollback 시도
- target이 current, 직접 predecessor, 더 오래된 ancestor
- 존재하지만 다른 history branch의 publication
- predecessor cycle 또는 중간 row 누락
- target child set이 비어 있거나 candidate membership과 다름
- 같은 expected version의 publish와 rollback 경합

### 실패 조건

- pointer version을 감소시킴
- target publication을 current로 직접 연결함
- existing publication row를 수정함
- schedule ID만 같다고 child evidence 차이를 무시함
- non-ancestor나 current 자신을 허용함
- audit 없이 새 current를 만듦

### 필요한 제약사항

- publication row와 child snapshot은 immutable이라고 가정하되 검증한다.
- traversal은 history 길이에 선형이고 visited set은 `O(h)` 공간을 사용한다.
- 긴 history 최적화가 필요하면 ancestor index/materialized path를 고려하되 append-only 의미를 유지한다.

## 구현 후 자가 검증

- [ ] direct predecessor와 오래된 ancestor가 허용되고 current/non-ancestor는 거부되는가?
- [ ] cycle과 missing predecessor에서 무한 루프 없이 실패하는가?
- [ ] 새 generation이 단조 증가하고 supersedes는 rollback 전 current인가?
- [ ] rollback target은 실제 복구 대상 과거 publication인가?
- [ ] target source snapshot과 child membership이 정확히 복제되는가?
- [ ] stale/unauthorized/invalid evidence에 partial history가 없는가?
- [ ] publish와 rollback 경합에서 CAS winner 하나만 생기는가?
- [ ] audit가 prior와 rollback target을 둘 다 정확히 기록하는가?

## 구현 후 설명할 것

- pointer rewind보다 append-only generation이 나은 이유
  - 모범답변: pointer를 과거 generation으로 되감으면 rollback 행위와 그 이후의 인과관계가 사라지고 version 단조성이 깨집니다. 새 generation을 추가하면 현재 상태를 복구하면서도 모든 승인·복구 이력이 시간순으로 남습니다.
- `supersedes`와 `rollback_target`의 의미 차이
  - 모범답변: `supersedes`는 새 publication 직전 current라서 history chain과 generation 연속성을 표현합니다. `rollback_target`은 어떤 과거 증거 상태를 복원했는지 나타내므로 rollback publication에서는 두 ID가 다를 수 있습니다.
- strict ancestor 제한이 branch 혼입을 막는 방식
  - 모범답변: current의 predecessor chain에서만 target을 허용하면 존재하기만 하는 다른 scope·분기 publication을 current history로 가져올 수 없습니다. current 자신도 제외해 의미 없는 rollback generation을 막습니다.
- visited set과 history 길이에 따른 복잡도
  - 모범답변: predecessor를 한 번씩 잠그므로 시간은 `O(h)`, cycle 검출용 visited set은 `O(h)`입니다. history가 커지면 ancestor index나 materialized path를 고려하되 append-only 원본 chain은 유지해야 합니다.
- target evidence exact copy가 “논리적으로 비슷한 복구”를 거부하는 이유
  - 모범답변: 같은 schedule ID처럼 일부 identity만 맞춰 재구성하면 당시 publication에 없던 duration이 섞일 수 있습니다. target의 source snapshot과 exact child membership을 검증·복제해야 역사적으로 게시됐던 상태를 그대로 복원합니다.

## 원본 확인 위치

- 파일: `travel_windows/publication.py`
- 함수: `_target_is_ancestor`, `rollback_flight_publication`, `_load_valid_candidate`, `_advance_pointer`
- DB guard: `travel_windows/migrations/0007_flight_review_lifecycle.py::travel_windows_guard_flight_publication_insert`, `travel_windows_guard_flight_pointer`
- Commit: `1b2a0df fix(db): close flight rollback and seal races`
- 관련 테스트: `travel_windows/tests/test_review_lifecycle.py::test_rollback_appends_generation_with_exact_historical_evidence`
- DB 거부 테스트: 같은 파일의 current target, duration closure, causal/audit identity 테스트
- 공통 entry/warning rollback: `reviews/publication.py::_rollback_publication_inner`

<a id="w13"></a>

---

# [reviews/publication.py::_failure_audit, _failure_outcome] W13 — 실패 transaction과 분리된 감사 기록 (A)

## 면접 질문

공통 publication 함수는 주 transaction이 실패해 rollback된 뒤 `_failure_audit`를 별도 durable transaction으로 실행하고, 이것마저 실패하면 원래 `STALE_POINTER`나 `NOT_AUTHORIZED` 대신 `AUDIT_UNAVAILABLE`을 반환합니다. 왜 실패 audit를 best-effort 부가 기능이 아니라 결과 계약의 일부로 취급했나요?

꼬리 질문:

1. 실패 audit를 원래 transaction 안에 insert하면 왜 의미가 없나요?
2. audit를 남기기 위해 subject/source/pointer를 다시 lock할 때 실패하면 subject 없는 audit를 허용하는 이유는 무엇인가요?
3. authorization failure와 publication failure action을 구분하는 이유는 무엇인가요?
4. `KeyboardInterrupt`, `SystemExit`, `GeneratorExit`를 문자열 code로 삼키지 않고 sentinel을 거쳐 chain 없는 새 예외로 다시 발생시키는 이유는 무엇인가요?

## 30초 모범 답변

주 transaction의 실패 기록을 같은 transaction에 넣으면 함께 rollback되므로 별도 durable transaction이 필요합니다. 이 시스템은 운영자 action의 성공뿐 아니라 실패도 감사 가능한 것을 invariant로 보므로 audit 작성 여부를 숨기지 않고, 기록할 수 없으면 `AUDIT_UNAVAILABLE`로 격상합니다. audit에는 폐쇄형 실패 code와 안전한 actor/subject identity만 넣습니다. 종료·취소 예외는 일반 업무 실패로 바꾸지 않고, 내부 frame이나 민감값 chain을 제거한 뒤 다시 발생시킵니다.

## 답변 핵심 키워드

- post-rollback audit
- separate durable transaction
- audit availability contract
- closed failure code
- sanitized process control
- minimal identity

## 꼬리 질문과 핵심 답변

### `AUDIT_UNAVAILABLE`이 원래 실패 원인을 가리는 단점은 없나요?

있습니다. 그러나 호출자에게는 “요청은 실패했고 그 실패조차 감사되지 않았다”가 더 높은 운영 위험입니다. 내부 지표에서 원래 분류를 별도로 세되, 공개 outcome은 감사 불능을 우선하는 정책입니다.

### 실패 audit transaction도 같은 module advisory lock을 잡는 이유는 무엇인가요?

성공 게시와 audit가 같은 subject/pointer identity를 동시에 읽을 때 일관된 순서를 유지하고, audit가 관찰한 scope를 고정하기 위해서입니다. lock order는 성공 경로와 같아야 합니다.

### actor 조회가 실패하면 unauthenticated로 기록해도 되나요?

정상적으로 인증되지 않은 actor는 null actor와 안전한 principal로 기록할 수 있지만, 인증됐다고 주장한 actor ID의 DB 조회가 불확실하면 허위 attribution을 만들지 말고 audit 자체를 unavailable로 봐야 합니다.

### 실패 audit에 exception message를 넣지 않는 이유는 무엇인가요?

DB/ORM 예외에는 SQL, 모델 값, request context가 섞일 수 있습니다. allowlist된 failure code와 결정적 input identity만 남겨 민감정보와 고카디널리티 로그를 막습니다.

### process-control 예외를 그대로 `raise`하지 않고 새 예외로 바꾸는 이유는 무엇인가요?

원 traceback과 chained callback frame에 raw body나 secret local이 남을 수 있습니다. 종류는 보존하되 새 chain 없는 예외로 다시 발생시키면 종료 의미와 비밀 비노출을 함께 지킬 수 있습니다.

## 백지 구현

### 구현 목표

원자적 업무 transaction의 실패를 rollback한 뒤 별도 transaction에서 최소·폐쇄형 audit로 남기고, audit 불능과 process-control 예외를 명확히 표면화하는 wrapper를 구현한다.

### 인터페이스

`run_audited_action(action_input: ActionInput, actor: ActorRef) -> ActionOutcome`

내부 함수:

- `perform_action_in_transaction(...) -> SuccessOutcome`
- `write_failure_audit(failure_code, safe_subject_id?, actor_ref) -> bool`

```python
import hashlib
import uuid


PROCESS_CONTROL = (KeyboardInterrupt, SystemExit, GeneratorExit)
AUDITABLE_FAILURES = {
    "STALE_POINTER", "NOT_AUTHORIZED", "INVALID_TARGET",
    "SOURCE_GATE_FAILED", "TRANSACTION_ABORTED",
}


class ActionRejected(Exception):
    def __init__(self, code):
        self.code = code


def _raise_sanitized(error):
    raise type(error)() from None


def _safe_actor(actor):
    if not getattr(actor, "is_authenticated", False):
        return ActorPrincipal("UNAUTHENTICATED", None)
    try:
        current = repository.lookup_actor(actor.id)
    except Exception:
        return ActorPrincipal("ACTOR_LOOKUP_UNAVAILABLE", None)
    if current is None:
        return ActorPrincipal("ACTOR_NOT_FOUND", None)
    return ActorPrincipal("AUTHENTICATED_OPERATOR", current.id)


def _audit_hash(code, action, module, subject_id, actor_kind, correlation_id):
    canonical = "\n".join(map(str, (
        "FAILURE_AUDIT_V1", code, action, module,
        subject_id or "", actor_kind, correlation_id,
    ))) + "\n"
    return hashlib.sha256(canonical.encode("utf-8")).hexdigest()


def write_failure_audit(failure_code, safe_subject_id, actor_ref,
                        *, module, action):
    if failure_code not in AUDITABLE_FAILURES or action not in ALLOWED_ACTIONS:
        return False
    correlation_id = uuid.uuid4()
    principal = _safe_actor(actor_ref)
    with repository.begin_durable():
        # rollback된 업무 row를 복원하지 않고, 존재가 안전하게 확인된 ID만 쓴다.
        subject_id = repository.safe_existing_subject(module, safe_subject_id)
        repository.insert_failure_audit(
            module=module,
            subject_id=subject_id,
            actor_id=principal.actor_id,
            actor_principal=principal.kind,
            action=action,
            outcome="FAILED",
            failure_code=failure_code,
            correlation_id=correlation_id,
            identity_sha256=_audit_hash(
                failure_code, action, module, subject_id,
                principal.kind, correlation_id,
            ),
        )
    return True


def _failure_outcome(code, action_input, actor):
    code = code if code in AUDITABLE_FAILURES else "TRANSACTION_ABORTED"
    try:
        audited = write_failure_audit(
            code,
            action_input.safe_subject_id,
            actor,
            module=action_input.module,
            action=action_input.action,
        )
    except PROCESS_CONTROL as error:
        _raise_sanitized(error)
    except Exception:
        audited = False
    return ActionOutcome(code if audited else "AUDIT_UNAVAILABLE")


def run_audited_action(action_input, actor):
    try:
        # perform 함수는 success audit도 이 transaction 안에서 생성한다.
        with repository.begin_durable():
            success = perform_action_in_transaction(action_input, actor)
        return success
    except PROCESS_CONTROL as error:
        _raise_sanitized(error)
    except ActionRejected as rejected:
        # 위 with block의 rollback/종료가 끝난 뒤에만 별도 audit를 시작한다.
        return _failure_outcome(rejected.code, action_input, actor)
    except Exception:
        return _failure_outcome("TRANSACTION_ABORTED", action_input, actor)
```

### 입력

- module과 subject ID
- actor reference
- expected pointer version 등 action 입력
- 업무 함수가 반환할 수 있는 폐쇄형 실패 종류

### 출력

- 업무 성공 outcome
- 감사된 업무 실패 outcome
- 실패 audit를 쓸 수 없으면 `AUDIT_UNAVAILABLE`
- process-control 예외는 return code가 아니라 sanitized exception으로 다시 발생

### 반드시 만족해야 할 조건

- 성공 audit는 업무 transaction 안에서 업무 row와 원자적으로 commit한다.
- 업무 실패 시 해당 transaction이 완전히 종료·rollback된 뒤 실패 audit transaction을 시작한다.
- 실패 audit에는 allowlist code, 고정 action, 안전한 actor principal, 가능한 subject ID, 새 correlation/transaction ID만 저장한다.
- subject/source를 안전하게 재검증할 수 없으면 임의 데이터를 채우지 않는다.
- failure audit transaction 실패를 숨기지 않고 `AUDIT_UNAVAILABLE`로 반환한다.
- process-control 예외는 종류를 보존하고 exception chain 없이 다시 발생시킨다.

### 경계 조건

- stale pointer, unauthorized, invalid target, source gate failure
- 주 transaction의 예상 밖 DB exception
- 주 transaction rollback 자체 후 audit 성공
- audit DB connection 실패
- actor가 unauthenticated, 삭제됨, 조회 중 오류
- `KeyboardInterrupt`, `SystemExit`, `GeneratorExit`

### 실패 조건

- 실패 audit를 주 transaction 안에 써 함께 rollback시킴
- audit 실패에도 원래 code만 반환해 감사 불능을 숨김
- exception text/traceback/raw subject 값을 audit에 저장함
- process-control 예외를 일반 `TRANSACTION_ABORTED` 결과로 삼킴
- audit를 위해 rollback된 업무 row를 다시 생성함

### 필요한 제약사항

- 업무 transaction과 failure audit transaction은 명확히 분리한다.
- failure code와 action은 enum/allowlist로 제한한다.
- audit hash는 안전한 identity 필드만 deterministic하게 결합한다.
- audit writer 자체는 무한 재시도하지 않는다.

## 구현 후 자가 검증

- [ ] 성공 시 업무 row와 success audit가 함께 commit되는가?
- [ ] 업무 실패 시 partial row가 rollback된 뒤 failure audit만 남는가?
- [ ] stale/authorization/source failure가 올바른 audit code/action으로 매핑되는가?
- [ ] audit writer 실패가 `AUDIT_UNAVAILABLE`로 드러나는가?
- [ ] unauthenticated actor와 actor lookup failure를 구분하는가?
- [ ] audit 데이터에 exception text·raw input·secret이 없는가?
- [ ] process-control 종류는 보존되고 traceback chain은 제거되는가?
- [ ] audit 실패를 다시 audit하려는 무한 재귀가 없는가?

## 구현 후 설명할 것

- 실패 audit가 반드시 별도 transaction이어야 하는 이유
  - 모범답변: 업무 transaction 안에 failure audit를 쓰면 업무 오류로 rollback될 때 audit도 사라집니다. 예외가 atomic block을 빠져나가 rollback이 완료된 다음 새 durable transaction을 열어야 실패 사실만 독립적으로 남습니다.
- 원래 failure보다 audit 불능을 우선 노출하는 정책 trade-off
  - 모범답변: 호출자는 원래 stale 같은 원인을 덜 자세히 알게 되지만, 시스템이 실패를 감사하지 못했다는 더 큰 운영 위험을 숨기지 않습니다. 원래 code는 외부 결과 대신 내부 metric에서 민감값 없이 추적할 수 있습니다.
- subject가 불확실할 때 최소 audit만 허용하는 기준
  - 모범답변: 안전한 module과 allowlist action, actor principal은 기록하되 subject는 별도 조회로 존재와 scope를 확인한 경우에만 ID를 넣습니다. rollback된 row나 raw 입력을 audit를 위해 재생성·복사하지 않습니다.
- 업무 예외와 process-control 예외를 구분한 이유
  - 모범답변: 예상 업무 실패와 일반 DB 예외는 폐쇄형 outcome으로 수렴할 수 있습니다. `KeyboardInterrupt`, `SystemExit`, `GeneratorExit`는 프로세스 제어 의도이므로 일반 실패로 삼키지 않고 종류만 보존한 sanitized exception으로 다시 올립니다.
- 결정적 audit identity hash가 제공하는 것과 제공하지 않는 것
  - 모범답변: 안전한 identity 필드의 변조·중복 비교와 재현 가능한 결합 형식을 제공합니다. 하지만 서명이나 외부 timestamp가 아니므로 누가 DB 전체를 변경했는지 증명하거나 audit row 삭제 자체를 막지는 못합니다.

## 원본 확인 위치

- 파일: `reviews/publication.py`
- 함수: `_failure_audit`, `_failure_outcome`, `_audit_identity_hash`, `_raise_sanitized_process_control`
- public wrappers: `publish_candidate`, `reject_candidate`, `rollback_publication`
- Commit: `8f85a69 feat(reviews): publish approved facts atomically`
- 관련 테스트: `reviews/tests/test_publication.py`, 특히 transaction failure/audit unavailable/process-control 테스트
- 운영 rehearsal: `operations/tests/test_publication_rollback_rehearsal.py`
- 대비 위치: flight 전용 성공 audit는 `travel_windows/publication.py::_create_audit`
