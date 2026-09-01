# Mobile Systems Evolution

## 1. 이력서용 프로젝트 설명

동일한 offline item tracker를 Android Kotlin과 React Native Android로 구현해 local-first 상태와 lifecycle 차이를 비교했습니다.  
Room·SQLite를 source of truth로 두고 화면 변경과 pending mutation을 원자적으로 기록하며, immutable dispatch identity·payload hash·base version으로 재전송 경계를 고정했습니다.  
ACK 반영과 dequeue의 원자성, version conflict·tombstone 보존, WorkManager와 native headless JS 기반 process-death 복구 및 network-loss cancellation을 구현했습니다.  
보존된 release 검증에서 두 트랙 모두 2,000개 row의 초기 읽기를 50개로 제한하고 네 개의 durable intent와 재시작 후 상태 보존을 확인했습니다.

## 2. 30초 프로젝트 소개

Offline item tracker를 Kotlin과 React Native Android로 구현했습니다. Room·SQLite를 source of truth로 두고 화면 변경과 pending mutation을 한 transaction으로 저장했습니다. WorkManager와 headless JS가 고정된 요청 identity로 process death와 응답 유실 뒤에도 같은 intent를 이어갑니다.

## 3. 2분 프로젝트 소개

모바일에서는 network 단절과 process death 뒤에도 사용자 변경을 지키는 것이 목표였습니다. 같은 offline tracker를 Kotlin과 React Native Android로 구현해 각각 Room·WorkManager와 SQLite·native headless JS를 사용했습니다. 두 구현 모두 local DB를 화면의 source of truth로 두고 변경과 pending mutation을 한 transaction으로 저장해 한쪽만 남는 상태를 막았습니다. 전송 전에는 mutation ID, payload hash와 base version을 고정합니다. ACK를 받으면 receipt, canonical item과 dequeue를 원자적으로 반영하고, 응답 유실은 같은 identity로 재시도합니다. 충돌 시에는 임의로 덮지 않고 pending intent와 tombstone을 남깁니다. background worker는 연결 조건과 재시도를 관리하며 React Native에서는 native 계층이 headless JS의 취소와 자원 정리까지 소유합니다. 보존된 최종 release 근거에서는 2,000개 row의 initial read를 50개로 제한한 뒤 네 intent의 내구성을 확인했습니다. 이번 검토에서는 제품 시나리오를 재실행하지 않고 checkpoint의 tag·commit·evidence 무결성만 검사했습니다. Android만 다뤘고 push·deep link·notification은 미구현이며, 이전 device scenario 전부를 최종 artifact에서 다시 실행한 것은 아닙니다.
