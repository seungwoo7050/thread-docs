# 릴리스 입력 정체성, 원자적 아티팩트, 증명

이 문서는 마스터 인덱스의 IM-01, IM-02, IM-19를 다룬다. 아래 함수 시그니처는 원본 구현을 복사한 것이 아니라 면접에서 핵심 invariant만 다시 구현하도록 축소한 연습용 인터페이스다.

<a id="im-01"></a>
## [Thread 1 / `fix(source): resolve archive remote refs`] Git ref를 불변 소스 정체성으로 해소하기

### 면접 질문

릴리스 입력으로 브랜치 이름을 받았을 때 로컬 브랜치와 `origin` remote-tracking branch를 어떻게 해소하고, 왜 최종 빌드는 detached commit에서 수행해야 합니까?

꼬리 질문:

- 로컬과 `origin`이 모두 존재하지만 서로 다른 커밋을 가리키면 어떻게 처리합니까?
- ref를 해소한 뒤 checkout 전에 브랜치가 이동하면 어떤 문제가 생깁니까?
- branch name, commit SHA, working tree snapshot 중 무엇을 evidence에 남겨야 합니까?

### 30초 모범 답변

브랜치 이름은 이동할 수 있으므로 릴리스 입력의 최종 정체성이 될 수 없습니다. 허용된 ref 공간을 명시적으로 조회해 하나의 커밋으로 해소하고, 모호하거나 서로 다른 결과가 나오면 실패시킨 뒤 그 커밋 SHA로 detached materialization을 수행해야 합니다. 해소와 materialization 사이에도 같은 SHA를 사용해야 경합으로 인한 source drift를 막을 수 있습니다. 편의성은 조금 줄지만 재현성과 attestation 신뢰도가 크게 올라갑니다.

### 답변 핵심 키워드

mutable ref, immutable commit, explicit lookup, ambiguity, fail closed, detached materialization, TOCTOU, reproducibility

### 백지 구현

**구현 목표**

요청한 ref에 대해 로컬 브랜치와 `origin` remote-tracking 후보를 입력받아 하나의 불변 커밋 정체성으로 해소한다. 실제 Git 명령 실행은 제외하고 해소 정책만 구현한다.

**인터페이스 또는 함수 시그니처**

```java
record RefCandidate(String fullRefName, String commitId) {}
record ResolvedSource(String requestedRef, String matchedRef, String commitId) {}

// 직접 구현
ResolvedSource resolveReleaseRef(
        String requestedRef,
        Optional<RefCandidate> localBranch,
        Optional<RefCandidate> originTracking) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 요청 ref, 로컬 후보, `origin` 후보
- 출력: 실제로 매칭된 ref 이름과 불변 commit identity

**반드시 만족해야 할 조건**

- 반환값에 symbolic branch 이름만 남기고 끝내지 않는다.
- 후보가 하나뿐이면 그 후보를 사용한다.
- 두 후보가 같은 commit identity를 가리키면 하나의 결과로 수렴한다.
- 두 후보가 서로 다른 commit identity를 가리키면 임의 우선순위를 두지 말고 모호성 오류로 처리한다.
- 입력 객체를 수정하지 않고 결과가 결정적이어야 한다.

**경계 조건**

- 두 후보 모두 없음
- 빈 요청 ref 또는 빈 commit identity
- 로컬과 `origin`이 같은 commit을 가리키는 경우
- full ref 이름은 다르지만 commit identity는 같은 경우

**실패 조건**

- 해소 불가
- 서로 다른 두 commit으로의 모호한 해소
- 유효하지 않은 후보 데이터

**필요한 제약**

- commit identity의 길이나 해시 알고리즘을 고정값으로 가정하지 않는다.
- 실제 materialization 단계는 반환된 commit identity만 입력으로 받아야 한다.

### 구현 후 자가 검증

- 정상 경로에서 로컬 전용, `origin` 전용, 양쪽 동일 commit이 모두 결정적으로 처리되는가?
- 두 후보가 다를 때 조용히 한쪽을 선택하지 않는가?
- 빈 값과 누락된 ref가 명확한 실패로 끝나는가?
- 반환값이 이후 단계에서 branch 재조회 없이 materialization에 사용될 수 있는가?
- 동일 입력에 동일 출력 또는 동일 오류가 발생하는가?
- 해소 결과가 mutable alias가 아니라 immutable identity인가?

### 구현 후 설명할 것

1. 로컬 우선 같은 편의 규칙 대신 모호성 실패를 선택한 이유
2. ref resolution과 checkout/materialization을 분리한 이유
3. 해소 후 branch가 이동하는 TOCTOU 문제를 막는 방법
4. evidence에 requested ref와 resolved commit을 함께 남길 가치

### 원본 확인 위치

- Thread 1 — 릴리스 입력 잠금과 소스 구체화
- 커밋: `ca6c5af4f18b — fix(source): resolve archive remote refs`
- 확인 가능한 컴포넌트: 로컬 브랜치, `origin` remote-tracking branch, detached source materialization
- 관련 Thread: 16, 17

---

<a id="im-02"></a>
## [Thread 2 / `build(jars): stage exact release artifacts atomically`] 완전한 아티팩트 세대를 원자적으로 게시하기

### 면접 질문

왜 서비스 JAR을 최종 디렉터리에 하나씩 복사하지 않고 임시 generation에서 정확한 집합을 검증한 뒤 원자적으로 게시해야 합니까?

꼬리 질문:

- 7개 중 6개만 생성된 상태에서 실패하면 소비자는 무엇을 보아야 합니까?
- 목적지에 이전 generation이 이미 있을 때 교체 실패를 어떻게 다룹니까?
- rename이 원자적이지 않은 파일시스템 경계라면 설계를 어떻게 바꿉니까?
- 두 게시자가 동시에 실행될 가능성이 있으면 어떤 소유권 규칙이 필요합니까?

### 30초 모범 답변

게시 단위는 개별 JAR이 아니라 검증된 전체 generation이어야 합니다. run-owned 임시 위치에 모두 생성하고 기대한 정확한 파일 집합인지 확인한 다음 같은 파일시스템 안에서 generation 포인터나 디렉터리를 원자적으로 교체해야 합니다. 검증이나 교체가 실패하면 이전 완전한 generation은 그대로 두고 임시 산출물만 정리합니다. 핵심 invariant는 소비자가 이전 완전본 또는 새 완전본만 보고 중간 상태는 절대 보지 않는 것입니다.

### 답변 핵심 키워드

all-or-nothing, exact set, staging generation, atomic rename, previous generation preservation, ownership, cleanup, same filesystem

### 백지 구현

**구현 목표**

후보 디렉터리의 파일 집합이 기대한 이름과 정확히 일치하는지 검증하고, 검증된 후보만 live generation으로 교체하는 게시기를 구현한다. 빌드 자체는 범위에서 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record PublicationResult(Path liveGeneration, Set<String> publishedArtifacts) {}

// 직접 구현
PublicationResult publishGeneration(
        Path candidateGeneration,
        Path liveGeneration,
        Set<String> expectedArtifactNames) throws IOException {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 임시 candidate generation, live generation 위치, 기대한 artifact 이름 집합
- 출력: 게시된 live 위치와 확인된 artifact 이름 집합

**반드시 만족해야 할 조건**

- 누락 파일과 예상 밖 파일을 모두 실패로 처리한다.
- 모든 검증이 끝나기 전에는 live generation을 변경하지 않는다.
- 교체 실패 시 기존 live generation을 보존한다.
- 성공 후 반환하는 집합은 실제 게시된 집합과 일치해야 한다.
- 원본의 7개 JAR 요구를 일반화해 기대 집합을 매개변수로 받는다.

**경계 조건**

- 기대 집합이 비어 있음
- candidate 디렉터리가 없음
- live generation이 처음 생성되는 경우
- live generation이 이미 있는 경우
- 파일 이름은 맞지만 디렉터리나 링크가 섞인 경우

**실패 조건**

- 누락·초과 artifact
- candidate 읽기 실패
- atomic 교체를 보장할 수 없는 경로
- 교체 도중 I/O 실패

**필요한 제약**

- candidate와 live는 원자적 이동을 지원하는 동일 파일시스템에 둔다.
- 외부 lock이 없다면 single-owner 실행을 전제로 명시한다.
- 임시 경로 정리는 게시 성공 여부와 독립적으로 수행할 수 있어야 한다.

### 구현 후 자가 검증

- 정확한 집합일 때만 live 상태가 바뀌는가?
- 하나가 빠지거나 하나가 더 있으면 이전 live가 그대로인가?
- candidate 검증 중 예외가 나도 live를 건드리지 않는가?
- 교체 실패 뒤 candidate와 backup의 상태를 추적할 수 있는가?
- 동시 게시를 허용하지 않는다면 그 제약이 인터페이스나 문서에 드러나는가?
- 파일 개수 검증만 하고 이름 중복·유형을 놓치지 않는가?
- 전체 파일 탐색의 시간·공간 복잡도를 설명할 수 있는가?

### 구현 후 설명할 것

1. 개별 파일 복사와 generation 교체의 원자성 차이
2. 정확한 집합 검증에서 누락뿐 아니라 초과 파일도 실패시키는 이유
3. 같은 파일시스템 제약과 cross-filesystem 대안
4. 이전 generation 보존과 임시 산출물 정리 순서
5. single-owner 또는 lock이 필요한 조건

### 원본 확인 위치

- Thread 2 — 격리 빌드와 원자적 아티팩트 게시
- 커밋: `f4a48d911ada — build(jars): stage exact release artifacts atomically`
- 확인 가능한 컴포넌트: run-owned Maven repository, 임시 generation, 정확히 7개 service JAR
- 관련 Thread: 8, 16

---

<a id="im-19"></a>
## [Thread 16 / `build(evidence): record locked release identities`] canonical release attestation 만들기

### 면접 질문

orchestration SHA, source SHA, lock identity, JAR hash를 왜 실행 중에 얻은 값으로 기록해야 하며, 이 값들을 어떤 방식으로 canonical attestation으로 묶어야 합니까?

꼬리 질문:

- 파일 경로나 생성 시각을 identity hash에 섞으면 어떤 문제가 생깁니까?
- JAR 목록 순서가 매번 달라져도 같은 attestation이 나오게 하려면 어떻게 합니까?
- 선언된 SHA와 실제 materialization 또는 파일 hash가 다르면 무엇을 신뢰해야 합니까?
- attestation에 secret이 들어가지 않는다는 것을 어느 단계에서 보장합니까?

### 30초 모범 답변

설정에 적힌 이름이 아니라 실제 실행이 사용한 orchestration SHA, materialized source SHA, lock identity, artifact hash를 기록해야 증거가 현실과 일치합니다. 같은 의미의 입력은 항상 같은 바이트 표현이 되도록 필드와 artifact 목록을 canonical order로 직렬화하고 그 결과를 검증 가능하게 저장합니다. 누락이나 선언값과 실측값의 불일치는 실패시켜야 하며, 비밀 값은 identity와 무관하므로 attestation 생성 전에 제외합니다. 이렇게 해야 릴리스가 어떤 소스와 잠금, 산출물로 만들어졌는지 독립적으로 재검증할 수 있습니다.

### 답변 핵심 키워드

runtime-derived identity, orchestration SHA, source SHA, lock identity, artifact hash, canonicalization, deterministic serialization, mismatch rejection, secret exclusion

### 백지 구현

**구현 목표**

릴리스 정체성 필드를 검증하고, artifact hash 순서와 입력 map 순서에 영향을 받지 않는 canonical byte representation을 생성한다. 실제 서명 알고리즘은 범위에서 제외한다.

**인터페이스 또는 함수 시그니처**

```java
record ReleaseIdentity(
        String orchestrationCommit,
        String sourceCommit,
        String lockIdentity,
        Map<String, String> artifactHashes) {}

// 직접 구현
byte[] canonicalize(ReleaseIdentity identity) {
    throw new UnsupportedOperationException("TODO");
}
```

**입력과 출력**

- 입력: 세 종류의 잠긴 identity와 artifact 이름별 hash
- 출력: 동일 의미 입력에 대해 항상 같은 canonical bytes

**반드시 만족해야 할 조건**

- 필수 identity가 하나라도 비면 실패한다.
- artifact 이름과 hash가 모두 비어 있지 않아야 한다.
- map iteration order와 무관하게 결과가 같아야 한다.
- 구분자 충돌이나 문자열 이어붙이기 모호성이 없어야 한다.
- secret, 로컬 절대 경로, 비결정적 timestamp를 identity 표현에 포함하지 않는다.

**경계 조건**

- artifact가 1개인 경우와 여러 개인 경우
- artifact 이름 정렬 순서가 입력마다 다른 경우
- 비ASCII artifact 이름
- 같은 artifact 이름의 중복 표현 가능성

**실패 조건**

- 누락 identity
- 중복 또는 모호한 artifact key
- 빈 hash
- canonical encoding 실패

**필요한 제약**

- 문자 encoding을 명시한다.
- 형식 version을 둘지 결정하고 설명한다.
- canonical bytes와 사람이 읽는 evidence 문서를 구분해도 된다.

### 구현 후 자가 검증

- map 삽입 순서를 바꿔도 bytes가 같은가?
- 필드 경계가 모호해지는 입력에서도 서로 다른 identity가 같은 bytes로 합쳐지지 않는가?
- 빈 필드가 조용히 직렬화되지 않는가?
- 비ASCII 입력이 환경별 기본 encoding에 영향을 받지 않는가?
- 로컬 경로나 실행 시각처럼 재현성을 깨는 값이 섞이지 않았는가?
- artifact hash를 하나 바꾸면 canonical 결과도 바뀌는가?
- 출력 크기와 정렬 비용을 설명할 수 있는가?

### 구현 후 설명할 것

1. 선언된 identity보다 runtime-derived identity를 우선하는 이유
2. canonicalization과 cryptographic signing의 역할 차이
3. artifact map 정렬과 encoding 명시가 필요한 이유
4. identity 필드와 운영 메타데이터를 분리하는 기준
5. Thread 15의 secret redaction gate와 결합되는 위치

### 원본 확인 위치

- Thread 16 — 의미 기반 릴리스 증명
- 커밋: `6184fc6137c — build(evidence): record locked release identities`
- 확인 가능한 컴포넌트: orchestration SHA, source SHA, lock identity, JAR hash
- 관련 Thread: 1, 2, 14, 15, 17
