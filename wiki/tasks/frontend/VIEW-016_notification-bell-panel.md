---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-016: 인앱 알림(Notification) 종모양 뱃지 패널"
labels: 'feature, frontend, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-016] 글로벌 사이드바/헤더 인앱 알림(Bell) 렌더링 컴포넌트
- 목적: 분석 완료, 워커 오류 등 시스템 수준의 주요 알림을 사용자가 즉각적으로 인지할 수 있도록 우상단 종모양 아이콘에 붉은 뱃지(Unread Count)를 띄우고 토글 다운 형태의 팝오버를 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#UI/UX Requirements`](#)
- 연결 라우터: [`API-002`](#) (SSE 엔드포인트)

## ✅ Task Breakdown (실행 계획)
- [ ] `radix-ui` 혹은 `headless-ui` 를 활용한 접근성 높은 Popover 컴포넌트 뼈대 작성
- [ ] `react-query` 를 이용한 알림 폴링(Polling) 혹은 SSE EventSource 리스너 훅 연결
- [ ] 안읽음(Unread) 건수 표시용 Notification Badge UI 구현
- [ ] 전체 읽음 처리(Mark All as Read) Mutation API 연결부 작성
- [ ] 글로벌 네비게이션바(VIEW-004) 우상단 헤더에 해당 컴포넌트 마운트

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 신규 알림 수신 및 뱃지 업데이트
- Given: 기존에 안읽은 알림이 2건 있는 상태로 유저가 로그인해 있음
- When: 백그라운드에서 문서 추출 완료 이벤트가 발생하여 SSE를 통해 새 알림 포맷이 도착함
- Then: 종모양 아이콘 우측 상단의 뱃지 숫자가 즉각적으로 '3'으로 변경되며, 아이콘 클릭 시 최상단에 새 이벤트 내역이 하이라이트된 상태로 노출된다.

Scenario 2: 전체 읽음 처리 
- Given: 안읽은 알림이 쌓여있는 팝오버를 열어놓은 상태
- When: 유저가 팝오버창 우상단의 '모두 확인' 버튼을 클릭함
- Then: 안읽음 뱃지가 즉시 사라지며, API 서버로 PATCH 요청이 넘어가 모든 알림의 `is_read` 가 True 로 동기화된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: SSE 연결이 끊어졌을 시 백그라운드에서 Exponential Backoff 를 통한 자동 재연결(p95 ≤ 300ms 지연 허용) 달성
- 안정성: 에러율 ≤ 0.5% 유지 (소켓/SSE 에러가 화면 전체의 White Screen을 유발하지 않아야함)
- 보안: 민감한 화주사 데이터(알람 내용 포함)가 타 세션에 전파되지 않도록 Context 엄격 분리

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 단위 테스트(Unit Test) 및 통합 테스트(Integration Test)가 추가되었고 통과하는가?
- [ ] SonarQube / Linter 등의 정적 분석 도구 경고가 없는가?
- [ ] API 명세서(Swagger 등)가 최신화되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-004` (글로벌 레이아웃 위치 확정), `API-002` (백엔드 송출기)
- Blocks: 없음
