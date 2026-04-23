---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ADMIN-007: Phase 2 테넌트 가입 초기 온보딩 플로우 기반 데이터 이관 트리거"
labels: 'feature, backend, phase:2, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-007] SaaS 가입 직후 초기화용 데이터 일괄 추출 마이그레이션(On-boarding) 트리거
- 목적: B2B 고객사가 카페24 API 식별키를 등록하자마자, 어제 하루치 데이터가 아닌 "과거 1년치 역사적 주문 데이터"를 한 번에 대량으로 다 끌어와서 AI 훈련 재료로 세팅해주는 초기 데이터 마중물(Background-Task) 엔진이다 (Phase 2 핵심 과제 중 하나).

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 연결 API: `ADMIN-002` (쇼핑몰 연동 키 저장 시나리오 직후)
- 대상 어댑터: `ADAPT-003` (카페24 등 데이터 소스)

## ✅ Task Breakdown (실행 계획)
- [ ] 가입 후 [과거 1년 데이터 동기화] 라는 어드민 API `POST /api/v1/admin/sync/historical` 신설
- [ ] 해당 요청을 받으면 `FastAPI BackgroundTasks` 나 `Celery` 큐에 타임아웃 걱정 없는 무거운 Bulk Import Job 을 밀어넣음
- [ ] 어댑터는 카페24 Rate Limit(초당 2건) 를 회피하기 위해 `time.sleep` 을 적절히 주면서 과거 365일에 해당하는 1년치 `ORDER` 테이블 적재 Loop 를 돌림
- [ ] Loop 가 무사히 성공하면 `API-002` (SSE 파이프) 를 이용해 "데이터 초기 세팅이 100% 완료되었습니다" 알람(Notification)을 송출

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: Rate Limit 우회 대량 쿼리 삽입
- Given: 방금 가입한 Tenant A 관리자가 "1년 동기화 시작" API 를 호출함.
- When: 서버는 202 Accepted (비동기 수행중) 만 던져주고 작업을 큐에 담음.
- Then: 뒤로 빠진 코어 루틴은 429 Too Many Requests 에러를 피하기위해 Paging 커서를 돌리며 수십 분간 조용히 DB `orders` 테이블에 과거 이력을 쏟아붓고 에러 없이 정상 종료된다.

Scenario 2: 데이터 뻥튀기(중복 삽입) 차단
- Given: 이미 초기 이관을 끝낸 A사가 이 버튼을 실수로 또 누름.
- When: 두 번째 호출이 발동함.
- Then: DB Upsert (Conflict on DO UPDATE) 룰에 의해 이미 채번된 오더 아이디는 덮어씌워지기만 할 뿐 중복된 row가 쌓이지 않아 데이터 결함(결측/왜곡)을 단단히 막아낸다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 응답시간 p95 ≤ 300ms 달성 (이 API 자체는 비동기 큐잉만 하고 바로 Return 하므로 300ms 통과. 본 연산은 백그라운드)
- 안정성: 에러율 ≤ 0.5% 유지 (중간에 서버가 추락해도 어디까지 폴링했는지 Cursor State 유지 필수)
- 보안: 비밀번호 평문 저장 절대 금지, 요청 페이로드 로깅 시 마스킹 처리 필수 (연동키 외부 유출 금지)

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 단위 테스트(Unit Test) 및 통합 테스트(Integration Test)가 추가되었고 통과하는가?
- [ ] SonarQube / Linter 등의 정적 분석 도구 경고가 없는가?
- [ ] API 명세서(Swagger 등)가 최신화되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `ADMIN-002` (인증 토큰 필수), `ADAPT-003` (어댑터 존재 필수)
- Blocks: AI 초기 세팅 훈련 엔진(`F1-002`) 가 해당 이관 전까진 동작할 수 없음.
