# Grocery Seasonality

## 1. 이력서용 프로젝트 설명

KAMIS 농산물 가격 데이터를 수집·검증해 승인된 최근·과거 가격만 공개하는 Django/PostgreSQL SSR 서비스를 구현했습니다.  
페이지·호출·응답 크기를 제한한 수집기와 엄격한 스키마·타입·중복 검증, canonical hash를 적용해 외부 응답을 재현 가능한 typed fact set으로 변환했습니다.  
검토·봉인된 불변 revision만 CAS로 활성화하고, PostgreSQL trigger와 deferred constraint로 append-only provenance와 current pointer 정합성을 DB에서도 강제했습니다.  
과거 후보 커밋에서 정적 검사와 859개 테스트, 로컬 브라우저·접근성·부하·백업/복원 검증을 남겼으며, 현재 HEAD 전체 회귀와 실제 운영 배포는 아직 확인되지 않았습니다.

## 2. 30초 프로젝트 소개

Grocery Seasonality는 KAMIS 가격을 그대로 노출하지 않고 검증·검토된 자료만 공개하는 Django/PostgreSQL 서비스입니다. 수집 응답의 범위와 스키마를 엄격히 제한하고, 승인·봉인한 불변 publication만 CAS로 활성화합니다. 특히 PostgreSQL trigger로 감사 이력과 현재 revision의 정합성을 DB에서도 방어한 점이 핵심입니다.

## 3. 2분 프로젝트 소개

이 프로젝트는 KAMIS 농산물 가격을 보여주되, 공급자 응답이 불완전하거나 재수집 결과가 달라져도 검증되지 않은 값이 공개되지 않게 하는 것이 목표였습니다. 구조는 제한된 수집, 엄격한 파싱, 사람 검토, 불변 publication, CAS 활성화, SSR 조회로 나눴고, 최근·월별·지역별·시장별 자료는 독립적으로 검토합니다. 핵심 결정은 원본 응답을 서비스 테이블에 곧바로 넣지 않는 것입니다. 수집기는 pagination, 호출·재시도·timeout·응답 크기를 제한하고, 파서는 정확한 필드·타입과 identity 중복을 검사해 canonical hash가 있는 fact set을 만듭니다. 원문 보관 권리가 불명확해 raw body 대신 hash, 길이, redacted receipt만 남겼습니다. 승인 뒤에는 예상 revision과 version을 비교하는 CAS로 pointer를 바꾸고, review·publication·activation 이력은 Django 트랜잭션뿐 아니라 PostgreSQL trigger와 deferred constraint로 불변성과 일관성을 강제했습니다. 공개 요청은 KAMIS를 재호출하지 않고 활성화·봉인된 revision만 읽으며, 잘못된 fact는 fail closed로 숨깁니다. 과거 vNext 후보 커밋에서는 Ruff, mypy, migration·Django check와 859개 테스트를 통과했고, 로컬 브라우저·Axe, 고정 10rps 부하, 백업·복원 fact parity와 제한된 live smoke 증거를 남겼습니다. 다만 이는 실서비스 성능·운영 복구의 증명이 아니며, 현재 HEAD 전체 회귀와 CI·배포 자동화는 확인되지 않았습니다. live smoke의 승인은 테스트용 자동 절차이지 실제 사람 승인 운영 데이터는 아닙니다.
