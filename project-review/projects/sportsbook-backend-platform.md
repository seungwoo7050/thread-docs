# Sportsbook Backend Platform

## 1. 이력서용 프로젝트 설명

베팅 접수·지갑·위험 판정·배당·정산·게이트웨이·관리 기능을 7개 Java 17/Spring Boot 서비스와 공통 Avro 계약으로 분리한 스포츠북 백엔드를 구현했습니다.  
베팅 접수를 Risk 예약→Wallet 출금→Risk 확정의 영속 상태 기계로 구성하고, 네트워크 타임아웃을 거절로 단정하지 않고 `PENDING` 상태에서 증거 조회·재시도·보상하도록 설계했습니다.  
Wallet에는 PostgreSQL 기반 멱등 결과와 복식부기 원장, transactional outbox를 적용하고, Settlement에는 순서가 뒤바뀐 이벤트의 catch-up과 결과 정정·owner-fenced lease 복구를 구현했습니다.  
Toxiproxy 장애 주입을 포함한 13개 교차 서비스 E2E 시나리오와 고정 커밋 기반 cold release gate를 구현해 실패·재전달·순서 역전 경계를 자동 검증 대상으로 만들었습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 베팅 접수부터 잔액·위험 판정·정산까지를 7개 서비스로 분리한 스포츠북 백엔드입니다. 핵심은 타임아웃을 실패로 단정하지 않는 것입니다. 베팅 단계를 데이터베이스에 남기고 불확실한 결과는 `PENDING`으로 보존한 뒤 Wallet 증거를 먼저 조회해 복구합니다. Wallet은 복식부기 원장과 outbox로 금액과 이벤트를 묶고, Settlement는 결과 정정을 lease 기반으로 처리합니다.

## 3. 2분 프로젝트 소개

응답 유실이나 이벤트 순서 역전에도 금액과 베팅 상태를 추적하는 스포츠북 백엔드입니다. Gateway, Betting, Wallet, Risk, Odds Feed, Settlement, Admin의 7개 Spring Boot 서비스와 공통 Avro 계약으로 구성했습니다.

핵심 결정은 베팅 접수를 동기 호출 묶음이 아니라 영속 상태 기계로 만든 것입니다. Betting은 Wallet 부수 효과 전에 Risk 예약 토큰을 저장하고, 재시도에도 같은 식별자와 토큰을 사용합니다. 타임아웃은 `PENDING`으로 남긴 뒤 Wallet 처리 증거를 먼저 조회해 필요한 단계만 이어가며, 확정 거절에는 진행 상태에 따라 Risk 예약 해제나 Wallet 환불을 수행합니다.

금액의 기준은 Wallet의 PostgreSQL로 한정했습니다. 멱등 키의 의미를 fingerprint로 검사하고, 계정 행 잠금 아래 잔액 변경, debit·credit 원장 쌍, 처리 결과와 outbox를 함께 기록했습니다. Settlement는 결과가 베팅보다 먼저 오면 저장 후 catch-up하고, 결과 정정은 이전·새 payout과 Wallet 증거를 고정한 revision plan으로 만든 뒤 database time과 owner-fenced lease로 복구합니다.

검증에는 Toxiproxy의 Risk 장애와 Wallet 응답 유실, 선행 이벤트·blocked 정정·revision 역순·동일 partition DLT를 포함한 13개 E2E 시나리오와 cold release gate를 구현했습니다. 다만 서비스별 `main` 저장소 구조와 과거 branch를 전제한 CI·materializer가 어긋나 clean clone 재현성에 간극이 있습니다. 저장소 참조 방식을 정리해야 합니다.
