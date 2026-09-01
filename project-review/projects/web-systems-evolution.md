# Web Systems Evolution

## 1. 이력서용 프로젝트 설명

HTTP endpoint monitoring 서비스를 Fastify·명시적 SQL과 Spring Boot·JPA 두 트랙으로 구현하고 Next.js UI와 PostgreSQL worker를 연결했습니다.  
`SKIP LOCKED` 기반 claim, committed lease와 owner-fenced completion으로 중복 실행과 worker crash 이후의 복구 경계를 설계했습니다.  
모든 DNS 응답과 redirect를 다시 검증하고 확인된 numeric address로 직접 연결해 SSRF와 DNS rebinding 경계를 코드로 통제했습니다.  
보존된 GitHub Actions 근거에서 두 트랙의 unit·PostgreSQL integration·browser·container job이 모두 통과했으며, readiness와 종료·로그·metric cardinality도 container 시나리오로 검증했습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 등록한 URL을 주기적으로 확인하는 monitoring 서비스를 Fastify와 Spring으로 각각 구현한 작업입니다. 실행권은 메모리가 아니라 PostgreSQL의 claim과 lease로 관리해 여러 worker와 crash 상황에서도 결과 확정 주체를 명확히 했습니다. 외부 호출은 모든 DNS 결과와 redirect를 검증한 뒤 승인한 IP에 직접 연결해 SSRF 경계를 통제했습니다.

## 3. 2분 프로젝트 소개

이 프로젝트는 endpoint CRUD보다 durable worker와 외부 URL 호출의 실패 경계를 다루는 데 초점을 맞췄습니다. 같은 product contract를 Fastify·TypeScript와 Spring Boot·Java로 구현했고, 둘 다 Next.js UI, PostgreSQL, API와 worker로 구성했습니다. 핵심 결정은 PostgreSQL을 실행 상태의 authority로 둔 것입니다. worker는 `FOR UPDATE SKIP LOCKED`로 대상을 claim하고 RUNNING 상태와 lease를 commit한 뒤 외부 I/O를 수행합니다. 완료 시 owner와 lease를 다시 검사해 자신의 실행만 확정하고, 만료된 실행은 회수합니다. 수동 실행에는 idempotency key를 적용해 재전송 중복도 막았습니다. 외부 호출은 URL과 모든 DNS 응답이 public address인지 검사하고 승인한 numeric address에 직접 연결해 DNS rebinding 창을 줄였습니다. redirect마다 이를 반복하고 timeout도 제한했습니다. 보존된 handoff에는 두 트랙의 unit·PostgreSQL integration·browser·container hosted CI가 해당 commit에서 성공한 기록이 있습니다. 이번 검토는 제품 테스트를 재실행하지 않고 그 기록과 Git ref·archive hash의 무결성만 확인했습니다. 99,000행 query plan도 고정 데이터 검증이지 일반 성능 보장은 아닙니다. Redis·Kafka·outbox 분산 단계는 미구현이고 공개 배포도 하지 않았습니다.
