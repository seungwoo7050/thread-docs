# Pong Pong

## 1. 이력서용 프로젝트 설명

Next.js, Fastify, WebSocket과 PostgreSQL로 실시간 Pong·matchmaking·tournament·chat을 제공하는 pnpm multi-package 애플리케이션을 구현했습니다.  
게임 진행 중 브라우저는 조작 input만 전송하고 서버가 bounded fixed-step simulation과 room 상태를 소유하도록 해 판정 기준을 단일화했습니다.  
input sequence, heartbeat, reconnect grace, latest-snapshot coalescing과 stable-key 기반 경기 결과 확정으로 실시간 연결과 영속화의 실패 경계를 다뤘습니다.  
공유 Zod HTTP·WebSocket contract와 unit·PostgreSQL integration·browser·fault-injection test 구성을 두고, 현재 checkout에서 production·load 설정 계약 13개를 재검증했습니다.

## 2. 30초 프로젝트 소개

Pong Pong은 브라우저가 조작만 보내고 서버가 fixed-step simulation과 승패를 authoritative하게 관리하는 실시간 웹 게임입니다. reconnect grace와 최신 snapshot만 남기는 backpressure를 적용했고, 경기 결과는 stable key로 PostgreSQL에 기록해 재시도에도 중복되지 않게 했습니다.

## 3. 2분 프로젝트 소개

Browser 연결, server simulation과 결과 저장이 따로 실패해도 일관성을 설명할 수 있는 웹 게임을 목표로 했습니다. 구조는 Next.js web, Fastify API·WebSocket hub, PostgreSQL repository와 shared contract package로 나눴습니다. 게임 중 browser는 paddle input과 sequence만 보내고 server가 fixed step으로 충돌·점수·승자를 계산합니다. event loop가 늦어져도 무한히 catch-up하지 않도록 한도를 두고 여러 room은 공통 scheduler에서 tick합니다. heartbeat로 단절을 감지해 reconnect grace 동안 session을 보존하고, 느린 client에는 snapshot을 쌓는 대신 미전송 값을 최신 상태로 교체합니다. HTTP와 socket payload는 같은 Zod schema로 검증하며 WebSocket ticket은 한 번만 소비합니다. 경기 결과는 stable key로 재시도하고 transaction과 unique constraint로 중복 전적을 막았습니다. 이번 검토에서는 dependency-free production·fault·load 설정 계약 13개가 통과했지만 전체 test suite와 실제 부하는 다시 실행하지 않았습니다. production mode에는 OAuth나 가입 흐름이 없어 새 session을 만들 수 없고, room ownership도 single process라 수평 확장을 주장하지 않습니다.
