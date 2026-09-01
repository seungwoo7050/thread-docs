# External-State Development Gap Audit
다음 두 입력을 사용하여 프로젝트의 기존 Development Thread 체계에서 누락된 **External-State Development Gap**을 분석하십시오.
## Inputs
### Git Repository
### Existing Development Thread Index
<THREAD_INDEX>:
````
````
기존 Thread별 학습 문서나 해설서는 이 단계의 입력으로 사용하지 마십시오.
이 작업의 목적은 기존 해설서의 내용을 평가하는 것이 아니라, Git 저장소 전체와 이미 확정된 Thread 구조를 기준으로 코드 밖에 존재할 수 있는 개발 단계를 독립적으로 발견하는 것입니다.

---
# 1. External-State Development Gap 정의
External-State Development Gap은 프로젝트의 소스 코드와 직접 연결되어 있지만, 그 개발 결과 자체가 Git의 최종 소스 코드나 일반적인 code diff 안에는 완전히 남지 않는 개발 단계입니다.
대표적인 예는 다음과 같습니다.
- database 또는 runtime resource의 실제 생성
- migration의 실제 실행
- seed의 실제 실행
- secret 또는 credential의 생성과 실행환경 주입
- 환경변수의 실제 runtime 구성
- OAuth application/provider의 외부 등록
- redirect URI의 외부 등록
- webhook endpoint의 외부 서비스 등록
- 외부 API credential 발급
- object storage, bucket, IAM 등의 외부 resource 구성
- DNS 또는 domain verification
- TLS certificate의 발급 또는 배치
- CI/CD platform의 secret 또는 environment 구성
- production/staging environment provisioning
- scheduler/cron의 외부 등록
- 실제 deployment 또는 runtime state 변경
- backup/restore 작업이 만들어내는 외부 상태 변화
단순한 설계 판단, 코드 리팩터링 이유, 실패했던 구현 시도 등은 이 분석의 대상이 아닙니다.
찾아야 하는 것은 **소스 코드와 연결되지만 소스 코드 그 자체가 아닌 개발 결과 또는 외부 상태 변화**입니다.
---
# 2. 증거 원칙
반드시 Git 전체 history와 실제 repository contents를 분석하십시오.
다음을 함께 확인할 수 있습니다.
- commit diff
- 현재 및 과거 source/configuration
- migration
- bootstrap/init script
- Docker/Compose configuration
- environment-variable reference
- CI/CD configuration
- deployment configuration
- database connection code
- external-service integration code
- test/setup script
- backup/restore script
- infrastructure-related source artifact
그러나 repository가 실제 외부 작업의 수행 여부를 증명하지 못한다면 수행되었다고 서술하지 마십시오.
예를 들어 코드가 `DATABASE_URL`을 요구한다는 사실로부터 다음은 판단할 수 있습니다.
- 실행환경에 database connection information이 필요하다.
하지만 다음은 증명할 수 없습니다.
- 개발자가 실제로 언제 어떤 값으로 DATABASE_URL을 등록했다.
- 특정 production database에 실제로 migration을 수행했다.
따라서 항상 다음을 구분하십시오.
**Repository Evidence**
Git 또는 repository에서 직접 확인되는 사실.
**Required External Step**
repository의 구현을 실제로 성립시키기 위해 필요한 외부 단계.
**Unobserved Execution**
필요성은 확인되지만 실제 수행 여부·시점·값·정확한 절차는 repository만으로 확인할 수 없는 부분.
실제 secret 값, credential 값 등은 추정하거나 복원하지 마십시오.
---
# 3. 기존 Thread와의 관계 판정
발견된 각 Gap을 다음 세 종류 중 하나로 분류하십시오.
## A. EXISTING_THREAD
특정 기존 Thread의 개발 관점을 실제 환경에서 완성하는 단계인 경우입니다.
Primary Owner가 되는 기존 Thread 하나를 지정하십시오.
다른 Thread와도 관계가 있다면 Related Threads로만 기록하십시오.
동일 내용을 여러 Thread에 중복 소유시키지 마십시오.
## B. NEW_THREAD
기존 Thread 어느 하나의 부수 단계가 아니라, 여러 구성요소를 관통하거나 자체적인 상태·수명주기·실패조건·복구과정을 가지는 독립적인 개발 관점인 경우입니다.
단, **NEW_THREAD는 반드시 repository에서 관련 commit을 선정할 수 있어야 합니다.**
기존 Thread에 이미 사용된 commit을 다시 사용해도 되고 기존 Index에서 어느 Thread에도 사용하지 않은 commit을 사용할 수도 있습니다.
외부 상태 자체가 diff에 남지 않는다는 이유만으로 commit이 없어도 되는 것은 아닙니다.
NEW_THREAD를 제안하려면 해당 관점을 구현·요구·연결시키는 대표 commit들을 반드시 선정하십시오.
## C. PROJECT_LEVEL_EXTERNAL_STEP
학습에는 필요하지만 다음 조건 중 하나에 해당하는 경우입니다.
- 특정 Thread에 자연스럽게 귀속되지 않는다.
- 독립적인 Thread로 만들 만큼의 개발 관점이 아니다.
- repository에서 새로운 Development Thread의 근거가 될 commit 집합을 선정할 수 없다.
이 경우 억지로 Development Thread를 생성하지 마십시오.
프로젝트 수준의 External Development Step으로 분류하십시오.
---
# 4. 신규 Thread 판정 기준
단순히 여러 Thread에서 사용하는 환경변수라는 이유만으로 새로운 Thread를 만들지 마십시오.
NEW_THREAD는 최소한 다음과 같은 독립성이 있어야 합니다.
- 자체적인 lifecycle이 있다.
- 여러 단계가 연결된 하나의 개발 문제를 형성한다.
- 실패 또는 복구 조건이 별도로 존재한다.
- 여러 subsystem을 관통하는 중요한 시스템 관점이다.
- 이후 유사한 프로젝트에서 독립적으로 학습할 가치가 있다.
그리고 반드시 해당 관점을 뒷받침하는 representative commits를 선정할 수 있어야 합니다.
관련 commit을 하나도 선정할 수 없다면 NEW_THREAD가 아닙니다.
---
# 5. 기존 Thread를 다시 정의하지 마십시오
Existing Thread Index는 이미 확정된 1차 Development Thread 체계입니다.
External-State 관점을 추가한 결과 명백하게 독립적인 새로운 관점이 발견될 경우에만 NEW_THREAD를 제안하십시오.
기존 Thread의 commit 구성이나 제목을 사소한 이유로 다시 최적화하지 마십시오.
이 작업은 기존 Thread 재분석이 아니라 **External-State Gap Audit**입니다.
---
# 6. 후속 문서 작성을 위한 Evidence Packet
Gap 목록만 출력하지 마십시오.
후속 세션이 전체 repository에 접근하지 않고도 판단할 수 있도록, 영향을 받는 각 기존 Thread와 각 신규 Thread에 대해 별도의 **Evidence Packet**을 작성하십시오.
Evidence Packet에는 프로젝트 전체 정보를 넣지 말고 해당 Gap을 이해하는 데 필요한 최소한의 자료만 포함하십시오.
각 Packet에는 다음이 포함되어야 합니다.
### Thread Identity
- Existing Thread 또는 Proposed New Thread
- Thread 번호 또는 임시 ID
- 한국어 제목
- English title
### Gaps
이 Packet이 다루는 Gap ID들.
### Repository Evidence
각 Gap의 근거가 되는:
- commit SHA
- commit subject
- 관련 파일
- 필요한 source/configuration 내용
- 필요한 경우 해당 commit의 관련 diff
관련 없는 파일이나 diff는 포함하지 마십시오.
### External Development Steps
repository evidence로부터 요구됨을 확인할 수 있는 코드 밖의 개발 단계.
### Code Connection
각 external step이 어떤 코드/configuration/runtime behavior와 연결되는지.
### Evidence Boundary
각 항목을 다음으로 명확히 구분하십시오.
- Directly observed in repository
- Required/inferred from repository
- Actual execution not observable from Git
### Ordering
학습자가 이해하기 가장 자연스러운 개발 순서.
실제 수행 이력이 증명되지 않는 external step을 마치 실제 chronological history인 것처럼 배열하지 마십시오.
필요한 경우 "conceptual execution order"라고 명시하십시오.
---
# 7. 신규 Thread용 Source Packet 특별 규칙
NEW_THREAD를 제안하는 경우 Evidence Packet은 이후 해당 Thread의 학습 문서를 작성하는 **유일한 project-specific source**로 사용할 수 있어야 합니다.
따라서 반드시 다음을 포함하십시오.
- representative commit 목록
- 각 commit이 이 Thread에서 중요한 이유
- Thread와 직접 관련된 commit diff
- 필요하다면 final-state configuration/source excerpt
- repository에서 직접 확인되는 사실
- repository로부터 요구됨을 확인한 external steps
- 확인할 수 없는 실제 실행 정보
그러나 해설서 자체를 미리 작성하지 마십시오.
Packet은 **근거 자료**여야 하며 학습 문서가 되어서는 안 됩니다.
---
# 8. 출력 구조
다음 구조로 결과를 작성하십시오.
## Part I — Gap Index
각 Gap에 대해:
- Gap ID
- 짧은 이름
- Classification:
  - EXISTING_THREAD
  - NEW_THREAD
  - PROJECT_LEVEL_EXTERNAL_STEP
- Primary Owner
- Related Threads
- Repository Evidence 요약
- Required External Step 요약
- 실제 수행 여부 확인 가능성
- Documentation Action
## Part II — Existing Thread Supplement Packets
External-State Gap이 발견된 기존 Thread에 대해서만 Packet을 작성하십시오.
Gap이 없는 Thread는 출력하지 마십시오.
## Part III — Proposed New Thread Packets
NEW_THREAD 판정을 받은 항목에 대해서만 작성하십시오.
각 신규 Thread에는 representative commits를 반드시 포함하십시오.
## Part IV — Project-Level External Steps
Thread로 만들지 않은 중요 external steps만 정리하십시오.
---
# 9. 중요 제한
- 기존 Thread별 학습 문서를 재작성하지 마십시오.
- 일반적인 웹 개발 지식을 근거로 프로젝트에 존재하지 않는 작업을 추가하지 마십시오.
- "보통 이렇게 한다"는 이유만으로 Gap을 만들지 마십시오.
- repository에서 구체적인 필요성을 확인할 수 있는 항목만 채택하십시오.
- 실제 수행되지 않았을 가능성이 있는 작업을 과거 사실처럼 작성하지 마십시오.
- secret 값이나 credential을 추정하지 마십시오.
- commit 수를 맞추기 위해 NEW_THREAD를 만들지 마십시오.
- 동일 Gap을 여러 Thread에 중복 설명하지 말고 Primary Owner를 지정하십시오.
목적은 기존 학습 자료를 뒤엎는 것이 아니라, **Git/source 중심 학습으로 인해 빠질 수 있는 실제 시스템 성립 과정을 최소한의 추가 문서로 보완하는 것**입니다.
다음 요청에서 <REPOSITORY> 와 <THREAD_INDEX> 를 입력하겠습니다.
해야할 작업을 정확하게 인지하고 대기하세요.