# 개발자 기술면접 워크북 06 — 쿼리 계획과 프로덕션 운영 경계

이 문서는 기능이 맞게 동작하는 것에서 한 단계 더 나아가, 실제 데이터 분포에서 쿼리 구조를 검증하고 프로세스의 생존·서비스 가능 상태·관측 가능성을 분리하는 문제를 다룬다. 단일 실행 시간이나 로그 양을 성과로 삼지 않고, 반복해서 유지해야 할 구조적 조건과 bounded cardinality를 중심으로 준비한다.

---

<!-- coverage: SA-19 -->
<a id="sa-19"></a>
## [Thread 13 / `test(e20): verify the fixed skewed history plan`] 실행계획을 구조로 검증하고 불필요한 인덱스를 만들지 않기

### 면접 질문

99,000개의 CheckRun이 한 Monitor에 편중된 고정 데이터셋에서 owner와 state로 제한한 최신 이력 20개를 조회했습니다. 실행 시간이 짧았다는 사실 대신 어떤 실행계획 조건을 검증해야 하며, 기존 인덱스가 조건을 만족할 때 새 partial 또는 covering index를 만들지 않은 이유는 무엇입니까?

꼬리 질문:

- `EXPLAIN ANALYZE`의 실행 시간만 임계값으로 두면 테스트가 불안정해지는 이유는 무엇입니까?
  - 모범답변: 실행 시간은 장비 성능, 캐시 상태, 동시 부하에 따라 달라집니다. 그래서 일반적으로 시간 임계값보다 기대 인덱스, 정렬 유무, 읽고 버린 행 수처럼 반복 가능한 구조를 검증하는 편이 안정적입니다.
- 페이지 크기는 20인데 SQL이 21행을 읽는 이유는 무엇입니까?
  - 모범답변: 이 프로젝트는 20개를 응답하고 한 행을 lookahead로 사용해 다음 페이지 존재 여부와 cursor 발급 여부를 판단합니다. 따라서 데이터베이스의 상한은 응답 크기보다 하나 큰 21행입니다.
- `Index Scan`이라는 이름만 확인하면 충분하지 않은 이유는 무엇입니까?
  - 모범답변: 다른 relation의 인덱스일 수도 있고, 조건과 정렬을 충족하지 못해 많은 행을 filter하거나 별도 Sort를 할 수도 있습니다. 정확한 인덱스 이름, relation, `Index Cond`, 생산·제거 행 수를 함께 봐야 합니다.
- 작은 Monitor 소유권 테이블의 sequential scan과 99,000행 CheckRun 테이블의 sequential scan을 같은 문제로 봐야 합니까?
  - 모범답변: 아닙니다. 작은 소유권 테이블은 sequential scan이 더 저렴할 수 있지만, 편중된 99,000행 이력 테이블의 scan은 조회량에 따라 비용이 커지므로 relation의 크기와 역할을 구분해 판정해야 합니다.
- 기존 인덱스가 충분한데도 covering index를 추가하면 어떤 write·storage·WAL 비용이 생깁니까?
  - 모범답변: insert와 인덱스 대상 column 변경마다 추가 인덱스 페이지를 갱신하고 WAL을 더 기록해야 하며, 저장 공간과 vacuum·cache 부담도 늘어납니다. 측정된 읽기 이득이 없으면 이 비용만 남습니다.
- 고정 데이터셋 한 번의 plan을 일반적인 운영 성능 보장이라고 말할 수 없는 이유는 무엇입니까?
  - 모범답변: 이 검증은 고정된 분포와 SQL에서 구조적 회귀가 없음을 증명할 뿐입니다. 운영의 데이터 분포, 통계, PostgreSQL 버전과 부하는 달라질 수 있으므로 실제 지표와 실행계획을 계속 관찰해야 합니다.

### 30초 모범 답변

실행 시간은 장비·캐시·동시 부하에 따라 흔들리므로 구조를 검증했습니다. 실제 production SQL과 바인딩이 frozen SQL과 같은지 먼저 확인하고, CheckRun 쪽에서 `(monitor_id, state, finished_at DESC, id DESC)` 인덱스를 사용하며 21행만 생산하는지, explicit Sort나 CheckRun sequential scan이 없는지, filter와 index recheck로 버리는 행이 없는지를 봤습니다. 21번째 행은 다음 cursor 존재 여부를 판단하는 lookahead입니다. 기존 인덱스가 이미 이 조건을 만족하므로 새 인덱스는 읽기 이득 없이 insert·상태 변경의 유지비와 저장 공간만 늘릴 수 있어 추가하지 않았습니다.

### 답변 핵심 키워드

production SQL parity, frozen bindings, skewed dataset, structural plan assertion, ordered composite index, limit plus one, no explicit Sort, no large-table sequential scan, rows removed, write amplification, WAL, no-index decision

### 백지 구현

#### 구현 목표

PostgreSQL의 `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` 결과를 재귀적으로 순회해 특정 relation의 실행계획이 면접에서 정한 구조적 조건을 만족하는지 평가한다. wall-clock 실행 시간은 판정에 사용하지 않는다.

#### 인터페이스 또는 함수 시그니처

```java
record RelationPlanAssessment(
        boolean expectedIndexUsed,
        boolean sequentialScanFound,
        boolean explicitSortFound,
        long producedRows,
        long rowsRemovedByFilter,
        long rowsRemovedByIndexRecheck,
        List<String> visitedNodeTypes,
        List<String> visitedIndexes) {}

final class ExplainPlanInspector {
    RelationPlanAssessment inspect(
            JsonNode explainDocument,
            String relationName,
            String expectedIndexName) {
        if (explainDocument == null || !explainDocument.isArray()
                || explainDocument.size() != 1
                || !explainDocument.get(0).path("Plan").isObject()) {
            throw new IllegalArgumentException("Malformed PostgreSQL JSON plan");
        }
        if (relationName == null || relationName.isBlank()
                || expectedIndexName == null || expectedIndexName.isBlank()) {
            throw new IllegalArgumentException("Relation and index names are required");
        }

        var result = new Accumulator();
        visit(explainDocument.get(0).get("Plan"), relationName, expectedIndexName,
                false, false, result);
        return new RelationPlanAssessment(
                result.expectedIndexUsed,
                result.sequentialScanFound,
                result.explicitSortFound,
                result.producedRows,
                result.rowsRemovedByFilter,
                result.rowsRemovedByIndexRecheck,
                List.copyOf(result.visitedNodeTypes),
                List.copyOf(result.visitedIndexes));
    }

    private void visit(
            JsonNode node,
            String relationName,
            String expectedIndexName,
            boolean sortOnPath,
            boolean relationContext,
            Accumulator result) {
        if (node == null || !node.isObject()) {
            throw new IllegalArgumentException("Malformed plan node");
        }

        String nodeType = node.path("Node Type").asText();
        if (nodeType.isBlank()) {
            throw new IllegalArgumentException("Plan node type is required");
        }
        result.visitedNodeTypes.add(nodeType);

        boolean currentSortOnPath = sortOnPath || nodeType.contains("Sort");
        String nodeRelation = node.path("Relation Name").asText();
        boolean relationNode = relationName.equals(nodeRelation);
        // Bitmap Index Scan에는 relation 이름이 없을 수 있으므로 바로 위 relation 문맥을 상속한다.
        boolean belongsToRelation = relationNode || (nodeRelation.isBlank() && relationContext);

        String indexName = node.path("Index Name").asText();
        if (!indexName.isBlank()) {
            result.visitedIndexes.add(indexName);
            if (belongsToRelation && expectedIndexName.equals(indexName)) {
                result.expectedIndexUsed = true;
            }
        }

        if (relationNode) {
            long loops = node.has("Actual Loops")
                    ? Math.max(0L, node.path("Actual Loops").asLong())
                    : 1L;
            result.producedRows += node.path("Actual Rows").asLong(0L) * loops;
            result.rowsRemovedByFilter += node.path("Rows Removed by Filter").asLong(0L) * loops;
            result.rowsRemovedByIndexRecheck +=
                    node.path("Rows Removed by Index Recheck").asLong(0L) * loops;
            result.sequentialScanFound |= "Seq Scan".equals(nodeType);
            // 관심 relation 위의 명시적 Sort만 이 조회 경로의 회귀로 기록한다.
            result.explicitSortFound |= currentSortOnPath;
        }

        boolean childRelationContext = relationNode
                || (nodeRelation.isBlank() && relationContext);
        JsonNode children = node.path("Plans");
        if (!children.isMissingNode() && !children.isArray()) {
            throw new IllegalArgumentException("Plans must be an array");
        }
        for (JsonNode child : children) {
            visit(child, relationName, expectedIndexName,
                    currentSortOnPath, childRelationContext, result);
        }
    }

    private static final class Accumulator {
        boolean expectedIndexUsed;
        boolean sequentialScanFound;
        boolean explicitSortFound;
        long producedRows;
        long rowsRemovedByFilter;
        long rowsRemovedByIndexRecheck;
        final List<String> visitedNodeTypes = new ArrayList<>();
        final List<String> visitedIndexes = new ArrayList<>();
    }
}
```

#### 입력과 출력

- 입력: PostgreSQL JSON explain 문서, 관심 relation 이름, 기대 인덱스 이름
- 출력: 해당 relation과 그 경로에서 관찰한 node·index·행 수·제거 행 수의 구조적 요약
- 출력은 판정 근거를 설명할 수 있도록 방문 node와 index 이름도 보존한다.

#### 반드시 만족해야 할 조건

- 최상위 배열과 `Plan` 객체를 올바르게 찾는다.
- 모든 중첩 `Plans`를 재귀적으로 순회한다.
- 관심 relation에 속한 node와 상위 explicit `Sort`를 구분해 기록한다.
- `Index Scan`, `Index Only Scan`, `Bitmap Index Scan`, `Bitmap Heap Scan`을 이름만으로 동일 취급하지 않고 relation·index 정보를 함께 본다.
- 기대 인덱스 사용 여부를 정확한 index 이름으로 확인한다.
- 관심 relation의 `Seq Scan`을 별도로 표시한다.
- `Actual Rows`는 `Actual Loops`와 함께 해석할 수 있게 총 생산 행 수를 계산하거나 두 값을 모두 명확히 보존한다.
- `Rows Removed by Filter`와 `Rows Removed by Index Recheck`가 없으면 0으로 처리한다.
- planning time과 execution time은 관찰하되 구조적 요구 조건으로 사용하지 않는다.
- malformed JSON이나 필수 `Plan` 누락은 정상 plan으로 간주하지 않는다.

#### 경계 조건

- 최상위가 배열 1개인 표준 PostgreSQL 형식
- relation 이름이 여러 node에 나타나는 bitmap plan
- `Actual Loops`가 2 이상인 nested loop 내부 scan
- `Sort`가 관심 relation 위에 있지만 다른 작은 relation에는 없는 경우
- index 이름은 있으나 다른 relation에 속한 경우
- `Rows Removed...` 필드가 생략된 경우
- `Limit` 아래에 index scan이 있는 경우
- parallel plan의 `Gather`·`Workers`가 포함된 경우
- 관심 relation이 plan에 전혀 없는 경우

#### 실패 조건

- JSON 문자열에서 단순 substring 검색만 하지 않는다.
- `Execution Time < N ms`만으로 통과시키지 않는다.
- `Index Scan` node가 하나라도 있으면 기대 인덱스가 쓰였다고 결론내리지 않는다.
- 전체 plan의 모든 sequential scan을 무조건 실패로 보지 않는다.
- 페이지 결과가 20개라는 이유만으로 database가 20행 이하를 읽었다고 추정하지 않는다.
- 테스트를 통과시키기 위해 planner 설정을 임의로 끄거나 데이터 분포를 바꾸지 않는다.

#### 필요한 제약

- 구현 대상은 explain 문서 분석기이며 SQL 실행기나 데이터 seed 코드는 제외한다.
- relation 하나와 기대 index 하나를 평가하는 20~25분 문제로 제한한다.
- 실제 판정 정책은 호출부에서 조합할 수 있도록 관찰과 정책을 분리한다.

### 구현 후 자가 검증

- [ ] 중첩된 모든 `Plans`가 누락 없이 순회된다.
- [ ] 다른 relation의 index 사용이 기대 index 사용으로 잘못 집계되지 않는다.
- [ ] 관심 CheckRun relation의 sequential scan을 탐지한다.
- [ ] explicit Sort가 관심 조회 경로에 있을 때 탐지한다.
- [ ] `Actual Rows × Actual Loops` 해석이 일관된다.
- [ ] filter와 index recheck 제거 행을 각각 집계한다.
- [ ] 21행 lookahead 조건을 호출부에서 검증할 수 있다.
- [ ] timing 필드 변화가 구조 판정을 바꾸지 않는다.
- [ ] malformed plan은 명시적으로 실패한다.
- [ ] 결과만 보고 추가 인덱스가 필요한지 단정하지 않고 write 비용까지 설명한다.

### 구현 후 설명할 것

1. 실행 시간 임계값 대신 구조적 조건을 택한 이유
   - 모범답변: 실행 시간은 환경 잡음에 민감하지만 인덱스 선택, explicit Sort 유무, 생산·제거 행 수는 조회가 확장되는 방식을 직접 보여 줍니다. 이 프로젝트의 고정 데이터셋 테스트도 wall-clock이 아니라 그 구조가 유지되는지를 판정합니다.
2. 실제 ORM SQL·바인딩과 EXPLAIN SQL을 동일하게 유지하는 방법
   - 모범답변: statement inspector로 애플리케이션이 생성한 SQL을 수집해 frozen SQL과 일치하는지 확인하고, 제한된 bind trace로 owner·monitor·state·limit 값을 대조합니다. 그 검증된 SQL과 같은 값으로 `EXPLAIN`을 한 번 실행해 다른 쿼리를 측정하는 오류를 막습니다.
3. limit+1이 pagination과 scan bound를 함께 증명하는 방식
   - 모범답변: 데이터베이스는 `limit+1`까지만 생산하고 애플리케이션은 초과 한 행을 응답에서 제외합니다. 초과 행이 있으면 다음 cursor를 만들 수 있고, plan의 `Actual Rows`가 21인지 확인하면 과도한 scan 없이 lookahead가 수행됐는지도 검증할 수 있습니다.
4. 작은 차원의 sequential scan과 큰 fact table scan을 다르게 보는 기준
   - 모범답변: planner는 relation 크기와 선택도에 따라 작은 테이블의 sequential scan을 합리적으로 선택할 수 있습니다. 이 프로젝트에서는 소유권 확인용 작은 relation이 아니라 99,000행 `check_runs`의 sequential scan과 정렬을 회귀 조건으로 봅니다.
5. 새 인덱스를 만들지 않는 것도 의도적인 성능 결정인 이유
   - 모범답변: 기존 `check_runs_history_state_order_idx`가 조건·정렬·21행 상한을 충족했으므로 추가 인덱스의 읽기 이득을 확인할 수 없었습니다. 이때 인덱스를 늘리지 않는 선택은 쓰기 증폭, WAL, 저장 공간과 유지보수 비용을 피하는 명시적 결정입니다.

### 원본 확인 위치

- Thread 13
- 커밋: `test(e20): verify the fixed skewed history plan`
- `backend/src/test/java/dev/evolution/monitor/HistoryQueryPlanTest.java`
- `backend/src/main/java/dev/evolution/monitor/MonitorStore.java`
- `MonitorStore.historyPage`
- `backend/src/test/java/dev/evolution/monitor/HistoryPaginationTest.java`
- `HistoryPaginationTest.SqlEvidence`
- `evidence/phase-1/E20/history.sql`
- 기존 인덱스 이름: `check_runs_history_order_idx`, `check_runs_history_state_order_idx`
- 관련 Thread: Thread 07의 keyset pagination과 cursor, Thread 03의 transaction 경계

---

<!-- coverage: SA-20 -->
<a id="sa-20"></a>
## [Thread 14 / `test(e24): freeze the operations and container contract`] liveness와 readiness를 프로세스 생존·권한 저장소·drain 상태로 분리하기

### 면접 질문

PostgreSQL 연결이 끊겼을 때 API와 worker의 liveness는 200을 유지하고 readiness는 503이 되도록 설계했습니다. liveness에 database 상태를 넣지 않은 이유와 worker readiness에 `acceptingClaims`, application readiness state, authority health를 모두 포함한 이유를 설명해 주세요.

꼬리 질문:

- database 장애 때문에 liveness까지 실패하면 orchestrator가 어떤 재시작 루프를 만들 수 있습니까?
  - 모범답변: 살아 있는 프로세스들을 같은 외부 장애 때문에 계속 재시작해 cold start와 연결 시도를 반복할 수 있습니다. database는 복구되지 않은 채 모든 인스턴스가 흔들리는 restart storm이 됩니다.
- 프로세스는 살아 있지만 신규 요청이나 claim을 받으면 안 되는 상태에는 무엇이 있습니까?
  - 모범답변: startup이 끝나지 않았거나 authority 저장소가 unavailable인 상태, Spring이 트래픽 거부 상태인 경우가 있습니다. worker는 graceful shutdown으로 drain을 시작해 `acceptingClaims=false`가 된 상태도 포함됩니다.
- graceful shutdown이 시작된 worker는 현재 작업을 마치고 있어도 readiness가 어떻게 되어야 합니까?
  - 모범답변: readiness는 즉시 DOWN이어야 합니다. 현재 소유한 작업의 정리는 계속하되 새 claim의 유입은 차단해야 lease ownership과 종료 시간이 예측 가능합니다.
- health endpoint에서 connection exception 메시지나 접속 정보를 반환하지 않은 이유는 무엇입니까?
  - 모범답변: 운영 판단에는 UP/DOWN이면 충분하며 예외 원문에는 hostname, JDBC 정보 같은 내부 구조가 포함될 수 있습니다. 이 프로젝트의 authority health도 세부 오류를 응답에 싣지 않고 실패를 DOWN으로만 접습니다.
- non-web worker에 별도 management listener를 둔 이유와 bounded executor가 필요한 이유는 무엇입니까?
  - 모범답변: worker는 애플리케이션 HTTP 서버가 없으므로 health와 허용된 metric만 제공하는 별도 관리 경계가 필요합니다. 고정 크기 thread와 queue는 probe 폭주가 worker의 업무 자원을 무한히 점유하지 못하게 합니다.
- readiness 확인 자체가 느리거나 무한히 쌓이면 어떤 운영 문제가 생깁니까?
  - 모범답변: orchestrator의 판단이 늦어지고 probe가 중첩되어 thread·메모리·connection을 소모할 수 있습니다. 결국 관리 plane 장애가 실제 업무 처리까지 압박하므로 확인은 짧고 자원 사용은 bounded여야 합니다.

### 30초 모범 답변

liveness는 프로세스를 재시작해야 하는지 판단하는 신호이고 readiness는 새 트래픽이나 작업을 받아도 되는지 판단하는 신호입니다. database가 잠시 내려갔다고 프로세스를 재시작하면 같은 외부 장애에 모든 인스턴스가 반복 재시작할 수 있으므로 liveness는 listener와 프로세스 생존만 봤습니다. readiness는 database authority가 실제로 읽히는지, Spring이 트래픽 수용 상태인지, worker가 shutdown drain 때문에 claim을 중단했는지를 함께 봅니다. 그래서 장애나 drain 중에는 503으로 라우팅·claim을 막되 프로세스는 살아서 복구를 관찰합니다. 응답은 UP/DOWN만 노출하고 내부 예외·연결 정보는 숨겼습니다.

### 답변 핵심 키워드

liveness, readiness, restart decision, traffic admission, authority health, acceptingClaims, ApplicationAvailability, graceful drain, dependency outage, bounded management plane, no private diagnostics

### 백지 구현

#### 구현 목표

프로세스 생존과 서비스 수용 가능 상태를 분리하는 순수 판정 함수를 작성한다. worker와 API가 공통 개념을 쓰되 worker의 claim 수용 상태를 추가로 반영할 수 있어야 한다.

#### 인터페이스 또는 함수 시그니처

```java
enum ProbeStatus { UP, DOWN }

record RuntimeSignals(
        boolean managementListenerRunning,
        boolean applicationAcceptingTraffic,
        boolean authorityAvailable,
        boolean workerRole,
        boolean workerAcceptingClaims) {}

record ProbeResult(int httpStatus, ProbeStatus status) {}

final class ProbePolicy {
    ProbeResult liveness(RuntimeSignals signals) {
        return result(signals.managementListenerRunning());
    }

    ProbeResult readiness(RuntimeSignals signals) {
        boolean ready = signals.managementListenerRunning()
                && signals.applicationAcceptingTraffic()
                && signals.authorityAvailable()
                && (!signals.workerRole() || signals.workerAcceptingClaims());
        return result(ready);
    }

    private ProbeResult result(boolean up) {
        return new ProbeResult(up ? 200 : 503, up ? ProbeStatus.UP : ProbeStatus.DOWN);
    }
}
```

#### 입력과 출력

- 입력: listener 생존, application traffic state, database authority, role, worker claim 상태
- 출력: HTTP status와 외부에 노출할 최소 상태
- exception 원문이나 접속 정보는 입력·출력 계약에 포함하지 않는다.

#### 반드시 만족해야 할 조건

- listener가 정상 동작하는 동안 일시적 authority 장애는 liveness를 내리지 않는다.
- authority 장애는 readiness를 내린다.
- application이 traffic 수용을 중단하면 readiness를 내린다.
- worker role에서 `workerAcceptingClaims=false`이면 readiness를 내린다.
- API role은 worker claim 신호에 의존하지 않는다.
- DOWN은 503, UP은 200으로 일관되게 매핑한다.
- 결과 body는 고정된 상태 값만 포함한다.
- 판정 함수는 database I/O를 직접 수행하지 않고 이미 수집된 신호만 조합한다.
- unknown·exception 상태는 readiness에서 fail closed로 처리할 수 있어야 한다.

#### 경계 조건

- startup 중 listener는 떴지만 application readiness가 아직 false
- 정상 실행 중 database만 중단
- database 복구 후 같은 process가 readiness를 회복
- shutdown 시작으로 worker claim 중단
- API role인데 worker flag가 false
- management listener stop 직전과 직후
- authority probe가 timeout·exception으로 끝난 경우
- 여러 신호가 동시에 false인 경우

#### 실패 조건

- 모든 dependency를 liveness에 넣어 재시작 폭풍을 만들지 않는다.
- readiness를 단순히 JVM이 살아 있다는 뜻으로 쓰지 않는다.
- worker가 drain 중인데 새 claim을 받을 수 있다고 보고하지 않는다.
- probe 응답에 stack trace, JDBC URL, hostname, credential을 넣지 않는다.
- health 요청마다 무제한 스레드나 대기열을 만들지 않는다.
- readiness 실패를 기존 작업의 결과 실패로 기록하지 않는다.

#### 필요한 제약

- pure function은 10~15분 안에 구현한다.
- 실제 management HTTP listener를 추가 구현한다면 GET-only, 고정 path allowlist, bounded thread/queue, `Cache-Control: no-store`까지만 확장한다.

### 구현 후 자가 검증

- [ ] database DOWN에서 liveness 200, readiness 503이다.
- [ ] database 복구 후 process restart 없이 readiness가 200으로 돌아온다.
- [ ] worker drain 중 readiness가 503이다.
- [ ] API role은 worker 전용 신호에 잘못 묶이지 않는다.
- [ ] listener 자체가 멈춘 상태는 liveness DOWN이다.
- [ ] startup·shutdown 경계에서 새 작업 수용 여부가 일관된다.
- [ ] 응답에는 내부 exception 세부정보가 없다.
- [ ] liveness와 readiness가 각각 어떤 운영 결정을 위한 것인지 설명할 수 있다.
- [ ] probe 계산이 application 업무 트랜잭션을 만들지 않는다.
- [ ] management transport의 thread·queue 상한을 명시할 수 있다.

### 구현 후 설명할 것

1. dependency 장애를 liveness에서 제외한 이유
   - 모범답변: liveness는 프로세스를 재시작하면 회복될 상태만 표현해야 합니다. 이 프로젝트에서는 database 장애 중에도 management listener가 살아 있으면 200을 유지해, 외부 dependency 장애를 불필요한 프로세스 재시작으로 확대하지 않습니다.
2. readiness를 traffic·claim admission control로 본 이유
   - 모범답변: readiness는 프로세스 존재 여부가 아니라 새 책임을 맡아도 되는지를 나타냅니다. 그래서 API의 traffic 수용 상태와 authority 가용성을 보고, worker에서는 여기에 `acceptingClaims`까지 더해 신규 유입을 제어합니다.
3. graceful shutdown과 lease 복구가 만나는 지점
   - 모범답변: shutdown이 시작되면 먼저 readiness와 `acceptingClaims`를 내려 새 lease를 얻지 않고, 이미 가진 작업만 제한된 시간 동안 마칩니다. 미완료 작업은 lease 만료 뒤 다른 worker의 recovery 대상이 되므로 drain과 복구가 ownership 경계를 공유합니다.
4. authority health 응답을 최소화한 보안 판단
   - 모범답변: 라우팅 결정에는 상태와 HTTP code만 필요합니다. 원본 구현은 `SELECT 1`의 성공 여부를 UP/DOWN으로만 변환해 connection 정보나 내부 exception이 관리 응답으로 새지 않게 합니다.
5. non-web worker management plane을 bounded resource로 만든 이유
   - 모범답변: worker에도 독립적인 liveness·readiness·metric 경로가 필요하지만 관리 요청이 업무 처리량을 잠식해서는 안 됩니다. 원본은 backlog 8, thread 2개, queue 8과 거부 정책을 사용해 관리 plane의 최대 점유량을 제한합니다.

### 원본 확인 위치

- Thread 14
- 커밋: `test(e24): freeze the operations and container contract`
- `backend/src/main/java/dev/evolution/monitor/AuthorityHealthIndicator.java`
- `AuthorityHealthIndicator.health`
- `backend/src/main/java/dev/evolution/monitor/WorkerManagement.java`
- `WorkerManagement.start`, `WorkerManagement.stop`, `WorkerManagement.isRunning`
- `backend/src/main/java/dev/evolution/monitor/CheckWorker.java`
- `CheckWorker.acceptingClaims`
- `backend/src/main/java/dev/evolution/monitor/WorkerProcess.java`
- `backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java`
- 운영 경로: `/ops/health/liveness`, `/ops/health/readiness`
- 관련 Thread: Thread 11의 graceful shutdown·lease recovery, Thread 09의 API/worker 분리

---

<!-- coverage: SA-21 -->
<a id="sa-21"></a>
## [Thread 14 / `fix: bound completed HTTP observation exports`] low-cardinality 메트릭과 안전한 구조화 로그 설계

### 면접 질문

`http.server.requests`의 `uri` tag에 실제 UUID path를 넣지 않고 고정 route template 또는 `UNMATCHED`만 허용했으며, inbound request-id를 그대로 신뢰하지 않고 새 UUID를 생성했습니다. 이런 제한이 필요한 이유와 queue·claim·recovery 메트릭을 실제 commit 이후에만 증가시킨 이유를 설명해 주세요.

꼬리 질문:

- `/api/monitors/{uuid}`를 raw path 그대로 tag로 쓰면 cardinality가 어떻게 증가합니까?
  - 모범답변: UUID마다 다른 `uri` tag 값과 별도 시계열이 생겨 요청 수에 비례해 cardinality가 계속 증가합니다. route template을 쓰면 모든 UUID 요청이 하나의 bounded 시계열로 합쳐집니다.
- method와 route를 allowlist 밖에서 각각 `OTHER`, `UNMATCHED`로 접는 이유는 무엇입니까?
  - 모범답변: 예상하지 못한 입력이 새로운 tag 값을 만드는 것을 막으면서도 정상 계약 밖 요청의 양은 관찰할 수 있기 때문입니다. method와 route를 서로 다른 고정 bucket으로 두면 원인도 구분할 수 있습니다.
- exception class나 error message를 metric tag로 넣으면 어떤 문제가 생깁니까?
  - 모범답변: class 집합은 늘어날 수 있고 message에는 사용자 입력과 식별자, 민감정보가 섞일 수 있어 시계열 수와 정보 노출 위험이 커집니다. 이 프로젝트는 HTTP status로 실패를 표현하고 `error` tag도 제거합니다.
- claim 시도 전에 counter를 증가시키면 database rollback 시 어떤 거짓 관측이 생깁니까?
  - 모범답변: 실제로 ownership이 저장되지 않았는데 committed claim이 늘어난 것처럼 보입니다. 원본은 transaction proxy가 성공해 `startNext`가 반환된 뒤에만 claim counter를 증가시킵니다.
- database가 없어 queue age를 계산할 수 없을 때 0과 `NaN`은 어떤 의미 차이가 있습니까?
  - 모범답변: 0은 queue가 비었거나 대기 시간이 사실상 없다는 유효한 값이고, `NaN`은 authority 장애로 측정 자체를 할 수 없다는 unknown입니다. 둘을 합치면 장애를 정상 상태로 오판할 수 있습니다.
- structured log에서 correlation은 유지하면서 password·cookie·CSRF token·URL 같은 private 값을 남기지 않으려면 어떻게 해야 합니까?
  - 모범답변: 서버가 생성한 request ID와 고정 event, method, route template, status, duration만 같은 schema로 기록합니다. body·header 전체와 raw URL, token류는 기본 필드에서 제외해 correlation key와 업무 비밀을 분리합니다.
- metric endpoint 자체는 왜 고정 이름 allowlist만 노출해야 합니까?
  - 모범답변: 임의 이름 조회를 허용하면 내부 metric 목록과 구현 정보를 탐색할 수 있고 노출 범위가 설정 변화에 따라 넓어질 수 있습니다. 원본 worker listener는 네 개의 운영 metric만 고정 집합으로 제공합니다.

### 30초 모범 답변

메트릭 tag 값은 가능한 조합 수가 시계열 수가 되므로 UUID path, exception text, 사용자 입력을 넣으면 cardinality와 메모리·저장 비용이 무한히 커질 수 있습니다. 그래서 method와 route는 고정 집합으로 정규화하고 high-cardinality key는 비웠습니다. request-id는 correlation용이지만 공격자가 긴 값이나 private 값을 주입하지 못하도록 서버가 생성했습니다. queue claim과 recovery counter는 시도가 아니라 durable state change를 뜻해야 하므로 transaction proxy가 성공해 commit된 뒤에만 증가시켰습니다. queue age를 읽을 권한 저장소가 없으면 빈 queue라는 0과 구분하도록 unknown을 표현하고, 로그에는 고정 event·role·process·request 메타데이터만 남겼습니다.

### 답변 핵심 키워드

metric cardinality, route template, allowlist normalization, OTHER, UNMATCHED, no high-cardinality tags, server-generated request id, post-commit counter, gauge semantics, unknown versus zero, structured logging, secret redaction, metric allowlist

### 백지 구현

#### 구현 목표

HTTP 요청을 bounded metric tags와 안전한 구조화 로그 이벤트로 변환하는 관측 정책을 작성한다. 프레임워크 계측 코드는 제외하고 값 정규화와 이벤트 구성만 구현한다.

#### 인터페이스 또는 함수 시그니처

```java
record RequestInput(
        String inboundRequestId,
        String method,
        String matchedRoute,
        String rawPath,
        int status,
        long durationMs) {}

record RequestObservation(
        String requestId,
        String methodTag,
        String routeTag,
        String statusTag,
        long durationMs,
        Map<String, Object> logFields) {}

interface RequestIdGenerator {
    String next();
}

final class BoundedHttpObservationPolicy {
    RequestObservation observe(RequestInput input, RequestIdGenerator ids) {
        Set<String> methods = Set.of(
                "GET", "HEAD", "POST", "PUT", "PATCH", "DELETE", "OPTIONS", "TRACE");
        Set<String> routes = Set.of(
                "/api/monitors",
                "/api/monitors/{id}",
                "/api/monitors/{id}/checks",
                "/api/monitors/{id}/checks/{checkId}",
                "/api/session",
                "/api/session/login",
                "/api/session/logout",
                "/api/session/csrf",
                "/ops/health",
                "/ops/health/**",
                "/ops/health/{*path}",
                "/ops/metrics",
                "/ops/metrics/{requiredMetricName}");

        String requestId = ids.next(); // inbound 값은 echo하지 않고 correlation key를 서버가 소유한다.
        String methodTag = input.method() != null && methods.contains(input.method())
                ? input.method()
                : "OTHER";

        String routeTag = input.matchedRoute() != null && routes.contains(input.matchedRoute())
                ? input.matchedRoute()
                : "UNMATCHED";
        if ("UNMATCHED".equals(routeTag)
                && input.rawPath() != null
                && !input.rawPath().contains("{")
                && routes.contains(input.rawPath())) {
            // Controller 이전에 끝난 security 응답도 고정 literal path만 예외적으로 허용한다.
            routeTag = input.rawPath();
        }

        String statusTag = input.status() >= 100 && input.status() <= 599
                ? Integer.toString(input.status())
                : "UNKNOWN";
        long durationMs = Math.max(0L, input.durationMs());
        Map<String, Object> logFields = Map.of(
                "event", "http_request",
                "request_id", requestId,
                "method", methodTag,
                "route", routeTag,
                "status", statusTag,
                "duration_ms", durationMs);

        return new RequestObservation(
                requestId, methodTag, routeTag, statusTag, durationMs, logFields);
    }
}
```

#### 입력과 출력

- 입력: inbound request-id, HTTP method, framework가 확인한 route pattern, raw path, status, duration
- 출력: bounded metric tag와 구조화 로그 필드
- raw path와 inbound correlation 값은 출력에 자동 포함하지 않는다.

#### 반드시 만족해야 할 조건

- request ID는 server-side generator에서 만든 값만 사용한다.
- method는 고정 allowlist에 있으면 그대로, 아니면 `OTHER`다.
- route는 고정 template allowlist에 있으면 그대로, 아니면 `UNMATCHED`다.
- framework route가 없는 조기 security 응답은 고정 literal path가 allowlist에 있을 때만 그 값을 사용할 수 있다.
- raw UUID path, query string, request body, header 전체를 metric tag로 쓰지 않는다.
- status는 100~599 범위만 문자열로 사용하고 나머지는 `UNKNOWN`이다.
- high-cardinality metric tags는 생성하지 않는다.
- log 필드는 `event`, `request_id`, `method`, `route`, `status`, `duration_ms` 같은 고정 schema만 사용한다.
- duration은 음수가 되지 않게 처리하거나 invalid 입력을 거부한다.
- password, cookie, authorization, CSRF token, idempotency key, URL, exception message는 기본 log field에 없다.
- 같은 종류의 UUID 요청 10개가 와도 tag 조합 수는 증가하지 않아야 한다.

#### 경계 조건

- 알려진 route와 UUID가 포함된 raw path
- controller 이전 security 401/403으로 matched route가 없는 경우
- allowlist에 없는 method
- allowlist에 없는 `/api/monitors/<uuid>/unknown` 경로
- inbound request-id가 매우 길거나 줄바꿈을 포함
- status 0, 99, 200, 404, 599, 600
- duration 0과 음수
- query string에 token이 포함된 요청
- 같은 template에 서로 다른 UUID 10개
- metric 조회 endpoint 자체의 요청

#### 실패 조건

- raw path를 정규식 한 번으로 UUID만 치환한 뒤 모든 나머지 경로를 허용하지 않는다.
- inbound request-id를 검증 없이 echo하거나 log하지 않는다.
- exception class·message·monitor ID·user ID를 low-cardinality tag에 넣지 않는다.
- claim·recovery counter를 transaction 성공 전에 증가시키지 않는다.
- database unavailable을 empty queue의 0으로 가장하지 않는다.
- 사용자가 요청한 임의 metric 이름을 그대로 조회·노출하지 않는다.

#### 필요한 제약

- route와 method allowlist는 immutable set으로 둔다.
- 구현 시간은 20분을 기준으로 한다.
- counter·gauge 등록 자체보다 값의 의미와 bounded cardinality를 우선한다.

### 구현 후 자가 검증

- [ ] 서로 다른 UUID 10개가 같은 route tag로 집계된다.
- [ ] unknown method는 `OTHER`, unknown route는 `UNMATCHED`다.
- [ ] inbound request-id가 출력 request-id에 영향을 주지 않는다.
- [ ] raw path·query·body·cookie·token이 log fields에 없다.
- [ ] status 범위 밖 값이 `UNKNOWN`으로 정규화된다.
- [ ] metric tag 집합 크기가 입력 cardinality에 비례해 늘지 않는다.
- [ ] rollback된 claim 시도는 committed claim counter에 반영되지 않는다.
- [ ] queue age 0과 authority unavailable을 구분할 수 있다.
- [ ] metric name endpoint가 고정 allowlist만 반환한다.
- [ ] 관측 코드의 실패가 application 결과를 새로 만들거나 바꾸지 않는다.

### 구현 후 설명할 것

1. metric cardinality가 운영 비용과 안정성에 미치는 영향
   - 모범답변: tag 값 조합마다 별도 시계열이 생기므로 cardinality가 커지면 수집기와 저장소의 메모리·인덱스·쿼리 비용이 함께 증가합니다. UUID나 message처럼 무한한 값을 넣으면 관측 시스템 자체가 장애 요인이 될 수 있습니다.
2. route template allowlist가 raw path 치환보다 안전한 이유
   - 모범답변: 정규식 치환은 예상하지 못한 식별자 형식이나 새 경로를 놓쳐 새로운 tag를 만들 수 있습니다. 고정 template allowlist는 계약에 등록된 값만 허용하고 나머지를 모두 `UNMATCHED`로 접어 cardinality 상한을 명확히 합니다.
3. server-generated request ID와 correlation의 trade-off
   - 모범답변: 외부 시스템이 보낸 ID를 그대로 이어 쓰는 편의는 포기하지만, 길이·개행·민감값 주입과 로그 오염을 막을 수 있습니다. 이 프로젝트는 응답에 새 ID를 돌려줘 클라이언트와 서버 로그의 correlation은 유지합니다.
4. post-commit metric이 durable invariant를 반영하는 방식
   - 모범답변: claim과 recovery counter는 시도 횟수가 아니라 database에 확정된 ownership 전이를 뜻합니다. transaction proxy가 정상 반환된 뒤 callback에서 증가시키므로 rollback된 변경은 지표에 포함되지 않습니다.
5. 0·unknown·실패를 gauge에서 구분하는 의미론
   - 모범답변: 0은 성공적으로 측정한 정상 값이고 unknown은 측정 authority가 없다는 뜻이며, 실패는 수집 경로 자체의 오류일 수 있습니다. 원본 queue age gauge는 database 예외를 `NaN`으로 바꿔 빈 queue의 0과 구분합니다.

### 원본 확인 위치

- Thread 14
- 커밋: `fix: bound completed HTTP observation exports`
- `backend/src/main/java/dev/evolution/monitor/HttpObservations.java`
- `HttpObservations.method`, `HttpObservations.route`
- `backend/src/main/java/dev/evolution/monitor/ProcessObservations.java`
- `ProcessObservations.recovered`, `ProcessObservations.storeUnavailable`, `ProcessObservations.request`
- `backend/src/main/java/dev/evolution/monitor/CheckQueue.java`
- `CheckQueue.oldestQueuedAgeSeconds`
- `backend/src/test/java/dev/evolution/monitor/HttpObservationsTest.java`
- `backend/src/test/java/dev/evolution/monitor/OperationsIntegrationTest.java`
- 운영 metric: `http.server.requests`, `check.queue.age`, `check.worker.active`, `check.claims`, `check.recoveries`
- 관련 Thread: Thread 10의 committed claim, Thread 11의 committed recovery, Thread 05의 private browser/session 값 보호
