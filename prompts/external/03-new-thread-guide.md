# New External-State Development Thread Guide
다음 Evidence Packet은 repository 전체를 분석한 별도의 External-State Gap Audit 단계에서 생성되었다.
이 Packet은 이 Development Thread의 학습 문서를 작성하기 위해 선별된 유일한 project-specific source of truth이다.
## New Thread Evidence Packet
<NEW_THREAD_EVIDENCE_PACKET>:
````

````
---
## 작업
Evidence Packet에 정의된 신규 Development Thread를 사람이 학습할 수 있는 완성된 학습 문서로 작성하라.
원격 repository를 추가로 조사하거나 Packet에 포함되지 않은 프로젝트별 사실을 사용하지 마라.
이 Thread는 source code의 구현 변화뿐 아니라, 해당 구현이 실제 시스템에서 성립하기 위해 필요한 external-state development step을 함께 다룬다.
문서를 읽은 학습자가 다음을 이해할 수 있어야 한다.
* 이 Thread가 해결하는 독립적인 개발 문제
* representative commit들이 해당 문제를 어떻게 단계적으로 구현하는지
* 각 코드 변화가 어떤 runtime/external requirement를 발생시키는지
* 코드 밖에서 어떤 상태의 생성·등록·주입·실행이 필요한지
* repository가 직접 증명하는 것과 실제 수행 여부를 확인할 수 없는 것의 차이
* 이후 유사한 시스템을 구축할 때 이 개발 관점을 어떻게 재현할 수 있는지
---
## Source Discipline
프로젝트별 사실은 Evidence Packet에 포함된 다음 자료만 사용한다.
* representative commit metadata
* selected commit diff
* 관련 source/configuration excerpt
* repository-observed behavior
* External-State Audit이 도출한 required external steps
Packet의 정보를 다음 세 범주로 엄격하게 구분한다.
### Directly Observed
Git/source/configuration에서 직접 확인되는 사실.
### Required / Inferred
해당 구현이 실제 시스템에서 동작하려면 필요한 외부 개발 단계.
### Unobserved Execution
필요성은 확인되지만 실제 수행 여부·시점·값·구체적인 실행 절차가 repository에서 확인되지 않는 부분.
Required/Inferred 정보를 개발자가 실제 수행한 historical event처럼 서술하지 마라.
Packet이 직접 증명하지 않는 다음 정보를 만들어내지 마라.
* 실제 실행 시점
* 실제 실행 명령
* 실제 secret/credential 값
* 실제 cloud/resource identifier
* 실제 dashboard 조작
* 실제 production 상태
* 실제 외부 시스템의 당시 상태
---
## Thread 유효성 검증
작성 전에 이 Packet이 실제 Development Thread를 구성할 수 있는지 검증한다.
다음 조건을 모두 만족해야 한다.
1. 독립적인 개발 관점이 존재한다.
2. representative commit이 존재한다.
3. 해당 commit의 관련 diff 또는 충분한 source evidence가 제공되어 있다.
4. external-state step이 해당 코드와 구체적으로 연결된다.
5. 단순한 프로젝트 사전조건이나 일반 운영 작업이 아니라 독립적으로 학습할 가치가 있다.
조건을 만족하지 못하면 문서를 작성하지 말고 다음만 출력한다.
`INVALID_AS_DEVELOPMENT_THREAD: reclassify as PROJECT_LEVEL_EXTERNAL_STEP`
---
## 설명 구조
commit을 단순 나열하거나 각각 독립적으로 요약하지 마라.
가능한 경우 다음 연결 구조를 중심으로 설명한다.
`기존 상태`
→ `commit에서 발생한 구현 변화`
→ `새롭게 생긴 runtime/external requirement`
→ `필요한 external-state development step`
→ `그 상태를 전제로 이어지는 다음 구현`
External-state step의 실제 수행이 확인되지 않는다면 chronological history처럼 표현하지 않고 conceptual execution order로 설명한다.
기존 commit-diff 기반 Thread 문서와 동일한 깊이의 기술적 설명을 유지하되, 외부 상태를 일반적인 배포 가이드로 과도하게 확장하지 마라.
Packet이 이 프로젝트에서 요구한다고 증명하는 범위만 설명한다.
---
## 출력
Evidence Packet이 유효한 Development Thread를 구성한다면 완성된 Thread 학습 문서만 출력하라.
