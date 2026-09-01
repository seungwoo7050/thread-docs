# Minishell (Small Shell)

## 1. 이력서용 프로젝트 설명

C99/POSIX로 인용부호·변수 확장, pipeline, `;`·`&&`·`||`, heredoc, redirection과 7개 builtin을 처리하는 제한된 shell을 구현했습니다.  
tokenizer와 parser, 실행 직전 expansion을 분리하고, 모든 heredoc을 입력 순서대로 준비한 뒤 조건 분기와 pipeline을 실행하도록 구성했습니다.  
상태를 바꾸는 단독 builtin은 부모에서 파일 디스크립터를 저장·복원해 실행하고, pipeline은 자식별 `fork`·`pipe`·`dup2`·`waitpid` 수명을 명시적으로 관리했습니다.  
로컬 테스트에서 parser 계약, 시스템 호출·할당 실패 주입, 자식/파일 디스크립터 수명과 512 KiB 입력 회귀를 검증했습니다.

## 2. 30초 프로젝트 소개

이 프로젝트는 shell 문법을 실행하는 C99/POSIX 프로그램입니다. tokenizer와 parser 뒤에 확장·heredoc·redirection 단계를 두고 pipeline을 `fork`와 `pipe`로 실행했습니다. `cd`와 `export` 같은 단독 builtin은 부모에서 실행하되 파일 디스크립터를 복원하고, 오류 경로에서도 자식과 descriptor를 모두 정리하도록 수명을 설계했습니다.

## 3. 2분 프로젝트 소개

목표는 완전한 POSIX shell보다 제한된 문법 안에서 문자열 해석과 프로세스·파일 디스크립터 수명을 일관되게 연결하는 것이었습니다. tokenizer가 인용 상태를 보존해 word와 operator를 나누고, parser가 명령, pipeline과 `;`, `&&`, `||` 구조를 만듭니다. heredoc은 입력 순서대로 먼저 수집하며 delimiter의 인용 여부로 본문 확장을 결정하고, 일반 인자와 redirection 대상은 실행 직전 환경과 종료 상태로 확장합니다. 단독 builtin을 부모에서 실행하는 이유는 `cd`, `export`, `unset`, `exit`의 상태가 현재 shell에 남아야 하기 때문입니다. 이때 stdin과 stdout을 복제해 redirection 뒤 복원합니다. pipeline은 명령별로 `fork`하고 `dup2`로 pipe를 연결한 뒤, 부모와 자식이 쓰지 않는 끝을 닫고 시작한 자식을 `waitpid`로 회수합니다. 중간 실패에서는 이미 연 자원과 자식을 정리하고 마지막 상태를 `$?`에 반영합니다. 로컬 `make test`에서 smoke, 시스템 호출·할당 실패, parser API, 자식·descriptor 수명과 512 KiB 입력 검사가 통과했습니다. 다만 job control, process group·foreground terminal 제어, `sigaction` 기반 prompt·heredoc 복구를 포함한 완전한 terminal signal 처리는 없으며 전체 POSIX 문법도 지원하지 않습니다.
