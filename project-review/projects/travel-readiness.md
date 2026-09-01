# Travel Readiness

## 1. 이력서용 프로젝트 설명

한국 일반여권 여행자의 가용 시간과 검수된 직항 운항·입국요건·여행경보 게시본을 결합해 여행 후보를 계산하는 Django SSR 서비스를 구현했습니다.  
외부 호출 전에 durable attempt를 기록하고, content-addressed artifact와 ordered parse lineage, schema fingerprint, sealed typed candidate로 공공데이터 provenance를 보존했습니다.  
worker는 후보 생성까지만 수행하고, 권한 있는 운영자가 PostgreSQL advisory/row lock과 CAS pointer 아래 승인·거절·새 generation rollback을 실행하도록 게시 경계를 분리했습니다.  
현재 HEAD에서 DB 비의존 focused test 48개와 deploy check를 재확인했으며, 전체 744개·browser·backup/restore 결과는 이전 completion report의 실행 기록으로 구분했습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 가용 시간에 맞는 직항 도시를 찾으면서 항공·입국·경보 데이터의 출처와 검수 이력을 추적하는 Django 서비스입니다. 수집 전 attempt부터 artifact, parse lineage, sealed candidate까지 증거를 남깁니다. worker는 후보까지만 만들고, 운영자만 한 트랜잭션에서 게시 pointer를 바꾸거나 새 generation으로 rollback합니다.

## 3. 2분 프로젝트 소개

공공데이터는 응답 구조가 바뀌거나 일부 호출이 실패할 수 있고, 여행 정보는 어떤 원문을 누가 검수해 게시했는지 추적할 수 있어야 한다는 문제에서 시작했습니다. 하나의 Django·PostgreSQL 시스템 안에서 항공, 입국요건, 여행경보의 evidence와 publication을 독립 모듈로 나눴습니다. 공개 검색은 외부 API를 호출하지 않고 세 모듈의 현재 게시본만 읽어 IANA timezone으로 현지 시각과 40시간 이상 체류 가능한 일정을 계산하며, 검색값과 결과는 응답 밖에 저장하지 않습니다. 수집기는 호출 전에 `STARTED` attempt를 커밋하고 성공 응답을 content-addressed artifact에 연결합니다. 여러 항공 페이지는 순서와 역할이 있는 parse lineage로 묶고 schema·identity 검증을 통과한 typed data만 seal합니다. worker는 후보까지만 만들며, 운영자 승인은 advisory lock, row lock과 pointer CAS를 한 durable transaction에 묶었습니다. rollback도 과거 row를 수정하지 않고 새 publication generation을 추가하고, DB trigger로 append-only 이력과 pointer 조건을 강제했습니다. 이번 감사에서는 DB 비의존 focused test 48개와 deploy check를 재실행해 통과를 확인했습니다. 전체 744개 test, browser, backup/restore와 local performance는 이전 completion report 기록이므로 현재 결과와 구분합니다. hosted CI와 공개 배포가 없고 runtime mtree pin 불일치와 browser harness의 절대 경로를 정리해야 합니다.
