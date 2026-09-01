이 세션에서 이전에 작성한 Development Thread 학습 문서는 이미 완성된 상태이다.
기존 문서를 다시 작성하거나 전체 내용을 재구성하지 마라.
별도의 repository-wide External-State Development Gap Audit에서 이 Thread와 관련된 다음 Evidence Packet이 생성되었다.
## External-State Evidence Packet
<EXTERNAL_STATE_PACKET>:
````

````
---
## 작업
먼저 이 세션에서 기존에 작성한 Thread 학습 문서와 당시 제공받았던 commit diff를 기준으로, 위 Packet이 지적하는 external-state 개발 과정이 기존 문서에 이미 충분히 설명되어 있는지 확인하라.
### 1. 이미 충분히 설명되어 있다면
새 문서를 작성하지 말고 다음만 출력하라.
`NO_SUPPLEMENT_NEEDED`
단순히 동일한 키워드가 언급되어 있다는 이유로 충분하다고 판단하지 마라.
학습자가 기존 문서만 읽고도 다음을 이해할 수 있어야 충분히 설명된 것이다.
* 코드가 실제 시스템에서 동작하려면 코드 밖에서 무엇이 성립해야 하는가.
* 어떤 external/runtime state가 실제로 만들어지거나 적용되어야 하는가.
* 그것이 기존 Thread의 코드와 어디에서 연결되는가.
* source/configuration의 정의와 실제 materialization/execution이 어떻게 다른가.
### 2. 실질적인 누락이 있다면
기존 문서를 수정하거나 재작성하지 말고, 기존 문서 뒤에 이어서 읽을 수 있는 별도의 `External-State Supplement` 문서만 작성하라.
기존 문서의 코드 설명을 반복하지 마라.
Supplement는 기존 Thread에서 누락된 external-state 개발 과정만 다룬다.
---
## 허용된 프로젝트별 근거
프로젝트에 관한 사실은 다음 두 source로 제한한다.
1. 이 세션에서 기존 Thread 문서를 작성할 때 제공되었던 commit diff와 자료
2. 위 External-State Evidence Packet
원격 repository를 별도로 조사하거나 Packet에 없는 프로젝트 사실을 추가하지 마라.
Packet에 기존 Thread의 원래 commit 집합에 포함되지 않은 commit이 등장하더라도, 그것은 이번 Supplement를 위한 evidence일 뿐이다.
그 commit을 기존 Development Thread Index에 자동으로 추가하거나 기존 Thread의 commit 구성을 변경하지 마라.
---
## Evidence Boundary
Packet의 정보를 다음 세 종류로 엄격히 구분하라.
### Directly observed
Git/source/configuration에서 직접 확인되는 사실.
### Required / inferred
해당 구현을 실제 시스템에서 성립시키기 위해 필요한 외부 단계.
### Unobserved execution
실제 수행 여부를 Git에서 확인할 수 없는 것.
Required/inferred 항목을 마치 개발자가 실제로 수행한 역사적 사건처럼 서술하지 마라.
Packet이 직접 증명하지 않는 다음 정보를 만들어내지 마라.
* 실제 수행 시점
* 실제 실행 명령
* 실제 secret 또는 credential 값
* 실제 resource ID
* 실제 production 상태
* 실제 dashboard 조작
* 실제 작업자가 사용한 절차
---
## 작성 목표
Supplement를 기존 Thread 문서 뒤에 읽었을 때 학습자는 다음까지 이해할 수 있어야 한다.
`코드와 configuration`
→ `그 코드가 요구하는 runtime/external condition`
→ `코드 밖에서 필요한 development step`
→ `실제 external state의 성립`
→ `다음 runtime 단계와의 연결`
다만 실제 수행 이력이 증명되지 않는 경우에는 이를 "실제로 수행했다"가 아니라 "이 구현을 동작시키려면 필요하다"는 관점으로 설명하라.
---
## 출력
누락이 없다면:
`NO_SUPPLEMENT_NEEDED`
누락이 있다면:
기존 Thread 문서를 대체하지 않는 완성된 Markdown `External-State Supplement` 문서만 출력하라.
