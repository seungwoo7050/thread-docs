workspace: `/Users/woopinbell/Desktop/working/project`
현재 workspace는 여러 소프트웨어 프로젝트를 모아둔 상위 폴더다.
목표는 **이직 포트폴리오 관점에서 프로젝트를 선별하고**, 가치가 있는 프로젝트에 대해서만 다음 3가지만 작성하는 것이다.
1. 이력서용 프로젝트 설명 3~4줄
2. 면접에서 "이 프로젝트를 설명해주세요"라는 질문에 대한 30초 답변
3. 같은 질문에 대한 2분 답변
---
# 1. 먼저 실제 프로젝트 경계를 판별하라
디렉터리 depth는 프로젝트 경계가 아니다.
다음 형태가 모두 존재할 수 있다.
* 1레벨 디렉터리 자체가 하나의 프로젝트
* 1레벨은 단순 그룹이고 2레벨 각각이 독립 프로젝트
* 1레벨 전체가 하나의 멀티모듈/멀티서비스 프로젝트이고 2레벨은 내부 서비스나 컴포넌트
* 작은 라이브러리나 exercise가 다른 프로젝트 아래 존재
따라서 단순히 1레벨 디렉터리를 프로젝트로 간주하지 마라.
README, `.git`, Makefile, CMakeLists.txt, package.json, pom.xml, Gradle 설정, Docker Compose, workspace 설정, 공통 설정, 내부 의존 관계, shared protocol/library, orchestration, 배포 구조, 각 디렉터리의 독립 실행 가능 여부 등을 확인해 **논리적인 프로젝트 경계**를 먼저 찾아라.
각 후보를 필요에 따라 다음과 같이 구분하라.
* INDEPENDENT_PROJECT
* PROJECT_GROUP
* MULTI_MODULE_PROJECT
* SERVICE_OR_COMPONENT
* SUPPORTING_LIBRARY
* EXERCISE_OR_SUBPROJECT
멀티서비스 시스템의 parent와 각 child service를 같은 프로젝트처럼 중복 평가하지 마라.
여러 서비스가 하나의 시스템을 구성한다면 가능한 경우 상위 시스템을 하나의 포트폴리오 프로젝트로 평가하고, 각 서비스는 그 프로젝트의 기술적 근거로 사용하라.
반대로 상위 디렉터리가 단순 분류용이고 하위 프로젝트들이 독립적이라면 하위 프로젝트를 각각 평가하라.
---
# 2. 전체 프로젝트를 먼저 상대평가하라
프로젝트별 설명을 작성하기 전에 모든 의미 있는 프로젝트를 먼저 비교하라.
분류:
## CORE
대표 포트폴리오 프로젝트.
기술적 깊이와 설계 판단이 충분하며 기술면접에서 상당한 대화를 이어갈 수 있다.
## STRONG SUPPORT
충분히 강한 프로젝트지만 CORE보다는 우선순위가 낮다.
## SUPPORTING
보조적으로 언급할 가치는 있지만 긴 프로젝트 소개나 2분 답변까지 준비할 필요는 없다.
## OMIT
너무 가볍거나, 다른 프로젝트와 중복되거나, tutorial/boilerplate 성격이 강하거나, 이직 포트폴리오에서 보여줄 가치가 낮다.
복잡하거나 코드가 많다는 이유만으로 높은 등급을 주지 마라.
작더라도 기술적으로 명확한 문제를 직접 해결한 프로젝트라면 높게 평가할 수 있다.
---
# 3. 프로젝트 종류에 맞는 기준을 적용하라
모든 프로젝트를 웹 프로젝트 기준으로 평가하지 마라.
C / 시스템 프로젝트라면 다음과 같은 항목을 본다.
* memory/resource ownership
* pointer discipline
* parsing
* syscall
* process
* signal
* file descriptor
* IPC
* synchronization
* concurrency
* event loop
* networking
* algorithm/data structure
* failure handling
* resource lifecycle
* performance constraint
C++ 프로젝트라면 추가로:
* RAII
* ownership model
* copy/move semantics
* STL
* polymorphism
* templates/generics
* exception/error strategy
* abstraction boundary
Frontend 프로젝트라면:
* component/module boundary
* rendering architecture
* server/client boundary
* hydration
* state ownership
* data/view-model boundary
* accessibility
* responsive design
* performance
* testing
* production build/deployment validation
Backend / Full-stack 프로젝트라면:
* API/domain boundary
* persistence
* transaction
* concurrency
* authentication/authorization
* consistency
* validation/error model
* frontend/backend contract
* testing
* deployment/operations
Graphics/Game/Infrastructure 등 다른 분야라면 해당 분야에 맞게 평가 기준을 스스로 조정하라.
---
# 4. 반드시 실제 구현을 근거로 판단하라
dependency에 기술이 있다는 이유만으로 해당 기술을 깊게 구현했다고 판단하지 마라.
가능한 경우 다음을 직접 확인하라.
* 실제 source code
* tests
* build scripts
* CI
* Docker/infrastructure
* README
* architecture/design docs
* 기존 interview docs
* 필요한 경우 git history
다음을 명확히 구분하라.
1. 직접 구현한 것
2. framework/library가 제공한 것
3. 단순 configuration
4. 문서에만 적혀 있고 코드로 확인되지 않는 것
5. 실제로 테스트/측정된 것
`production-grade`, `scalable`, `secure`, `high-performance`, `enterprise-ready`, `robust` 같은 표현은 명확한 근거 없이 사용하지 마라.
예:
나쁨:
"고성능 확장 가능한 서버를 구현했다."
좋음:
"non-blocking socket과 event multiplexing을 기반으로 connection/file descriptor lifecycle을 직접 관리하는 서버를 구현했다."
나쁨:
"production-grade frontend를 구축했다."
좋음:
"production build E2E, accessibility, bundle budget, Lighthouse 및 container runtime 검증을 CI에 연결했다."
---
# 5. 지시보다 과도한 작업을 하려고 시도하지 말아라
선별된 프로젝트에대해 아래 3가지만 작성한다.
1. 이력서용 프로젝트 설명 3~4줄
2. 면접에서 "이 프로젝트를 설명해주세요"라는 질문에 대한 30초 답변
3. 같은 질문에 대한 2분 답변
이 작업의 목적은 **기존 심층 면접 자료를 다시 만드는 것이 아니라 그 위에 채용용 진입점을 만드는 것**이다.
---
# 6. 전체 평가 결과
다음 파일을 생성하라.
`/Users/woopinbell/Desktop/working/thread-docs/project-review/00-PORTFOLIO-RANKING.md`
다음 표를 포함한다.
| Rank | Project | Type | Classification | Score / 100 | Best Target Roles | Strongest Engineering Signal | Main Weakness |
그리고 다음 섹션을 작성한다.
## Recommended Anchor Projects
최종 포트폴리오에서 가장 먼저 보여줄 프로젝트를 약 3~6개 선정한다.
## Strong Supporting Projects
## Supporting Projects
## Projects to Omit
## Redundancy Analysis
비슷한 역량을 보여주는 프로젝트가 여러 개라면 어느 프로젝트를 대표로 사용할지 정한다.
예를 들어 비슷한 네트워크 프로젝트가 여러 개라면 전부 보여주는 것보다 가장 강한 프로젝트를 대표로 선택한다.
## Role-oriented Selection
실제 프로젝트 구성이 뒷받침하는 경우에 한해 다음 직무별 프로젝트 우선순위를 제안한다.
* Frontend Engineer
* Full-stack Engineer
* Backend Engineer
* C/C++ / Systems Engineer
* General Software Engineer
---
# 7. CORE / STRONG SUPPORT에 대해서만 채용용 설명을 작성하라
CORE와 STRONG SUPPORT 프로젝트에 대해서만 다음 파일을 생성한다.
`/Users/woopinbell/Desktop/working/thread-docs/project-review/projects/<project-slug>.md`
SUPPORTING과 OMIT 프로젝트에는 긴 설명을 만들지 마라.
각 파일에는 **오직 다음 세 가지 본문만 작성한다.**
# `<Project Name>`
## 1. 이력서용 프로젝트 설명
한국어 기준 3~4줄.
다음을 우선한다.
* 무엇을 만들었는가
* 가장 중요한 설계/구현 결정
* 다른 프로젝트와 구별되는 기술적 특징
* 의미 있는 테스트/성능/배포/운영 작업이 있다면 포함
단순 기술 스택 나열문으로 만들지 마라.
면접관이 읽고 기술 질문을 할 만한 포인트가 자연스럽게 드러나야 한다.
---
## 2. 30초 프로젝트 소개
면접관이 다음과 같이 물었을 때 실제로 말할 답변을 작성한다.
"이 프로젝트에 대해 간단히 설명해주세요."
다음을 포함한다.
* 프로젝트의 목적
* 핵심 구조
* 가장 중요한 기술적 결정 1~2개
모든 기능을 나열하지 마라.
오히려 이 프로젝트에서 내가 가장 잘 방어할 수 있는 기술 주제를 자연스럽게 노출해 면접관이 그 부분을 질문하도록 만드는 답변이어야 한다.
실제로 소리 내어 말했을 때 약 30초 안에 들어가야 한다.
---
## 3. 2분 프로젝트 소개
30초 소개보다 깊게 설명하되 단순히 내용을 길게 늘리지 마라.
대략 다음 흐름을 따른다.
문제/목표
→ 전체 구조
→ 중요한 설계 결정
→ 핵심 구현
→ 테스트/검증/배포 등
→ 중요한 trade-off 또는 한계
실제 개발자가 자신의 프로젝트를 설명하는 말투로 작성한다.
마케팅 문구처럼 쓰지 마라.
---
# 8. 출력하지는 않지만 내부적으로 반드시 근거 검증을 수행하라
별도의 Evidence Map 문서를 만들 필요는 없다.
하지만 3~4줄 설명, 30초 답변, 2분 답변에 포함한 **모든 주요 기술 주장에 대해 내부적으로 실제 코드 또는 기존 기술 문서의 근거 위치를 확인한 뒤 작성하라.**
면접관이 각 문장에 대해:
"그거 정확히 어디서 구현했나요?"
라고 물었을 때 repository 안에서 실제 근거를 찾을 수 없는 문장은 약화하거나 삭제하라.
파일명이나 함수명을 내가 외워야 한다는 의미가 아니다.
목적은 **과장된 포트폴리오 문구가 생성되는 것을 막는 것**이다.
---
# 9. 가벼운 프로젝트를 억지로 부풀리지 마라
프로젝트가 실제로 작다면 솔직하게 SUPPORTING 또는 OMIT으로 분류하라.
작은 프로젝트에 다음과 같은 것을 억지로 만들지 마라.
* 거창한 architecture narrative
* 불필요한 2분 설명
* production/scalability 이야기
* 실제 범위를 넘어선 system design 주장
작지만 특정 기술 하나를 명확히 보여준다면 SUPPORTING 프로젝트로 남기는 것으로 충분하다.
---
# 10. 상대평가가 중요하다
각 프로젝트를 독립적으로 모두 "좋은 프로젝트"라고 평가하지 마라.
이 workspace 안의 프로젝트들을 서로 비교하라.
프로젝트 A와 B가 같은 역량을 보여주고 B가 명백히 더 강하면 A의 포트폴리오 우선순위를 낮춰라.
목표는 프로젝트 개수를 최대화하는 것이 아니다.
**채용 담당자에게 보여줄 소수의 강하고 서로 다른 프로젝트를 고르는 것**이다.
---
# 11. 최종 검토
모든 결과를 작성한 뒤 senior engineer 관점에서 다시 검토하라.
각 이력서 문장과 프로젝트 소개 문장마다 다음 질문을 적용한다.
"소스코드의 어디에 구현되어 있습니까?"
"직접 구현한 겁니까, framework가 해준 겁니까?"
"왜 그렇게 설계했습니까?"
"이 표현은 실제 측정/검증 없이 너무 강한 주장 아닙니까?"
근거가 약하면 표현을 낮추거나 제거하라.
이 작업의 최종 목적은:
**전체 프로젝트 선별 → 강한 프로젝트만 선택 → 이력서 3~4줄 → 30초 소개 → 2분 소개**
까지 완성하는 것이다.
