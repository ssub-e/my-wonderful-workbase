---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-002: 시스템 알림 및 진행상태 실시간 전송용 SSE(Server-Sent Events) 라우터"
labels: 'feature, backend, priority:medium'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-002] EventStream 을 이용한 클라이언트 단방향 실시간 알림 파이프
- 목적: 폴링(Polling)의 부하를 줄이기 위해, "PDF 추출 완료", "어드민에 의한 플랜 변경 강제 로그아웃" 등 백엔드에서 튀어나오는 비동기 이벤트들을 HTTP/1.1 의 `text/event-stream` 채널을 열어두고 프론트엔드(`VIEW-016`)로 즉각 Push 해주는 심장부 역할을 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md`](#)
- 연결 뷰: `VIEW-016` (사이드바 인앱 알람 컴포넌트)
- 관련 기술: FastAPI `EventSourceResponse` (sse-starlette)

## ✅ Task Breakdown (실행 계획)
- [ ] `pip install sse-starlette` 의존성 확보
- [ ] Redis Pub/Sub(`INFRA-002`) 채널에 `notifications_{tenant_id}` 형태의 토픽 리스너 등록
- [ ] FastAPI `GET /api/v1/notifications/stream` 라우터 작성 (Generator 기반 무한 루프)
- [ ] 연결이 끊어졌을 때를 대비해 클라이언트의 `Last-Event-ID` 를 받아 미수신 알람을 재발송하는 내결함성 코드(Fall-forward) 삽입

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: SSE 커넥션 수립 및 알람 Push 수신
- Given: 클라이언트가 로그인 후 `/api/v1/notifications/stream` 으로 GET 을 날려 HTTP 채널을 Holding 함.
- When: 백그라운드 워커에서 `Redis.publish("notifications_TENANT_A", json)` 을 쏨
- Then: 대기 중이던 FastAPI Generator 가 즉각 Yield 를 뱉으며, 클라이언트 브라우저의 Network EventStream 에 JSON 메시지가 도달해 프론트엔드가 종 모양 알람 뱃지 숫자를 올리게 한다.

Scenario 2: 인증 만료 끊김 방어
- Given: 미인증 유저(JWT 없음)가 스트림 채널을 열람 시도함
- When: 커넥션 핸드쉐이크 요청
- Then: 소켓을 아예 맺어주지도 않고 401 Unauthorized 로 차단하여 서버의 소켓 고갈(Socket Exhaustion DDOS)을 원천 차단한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 응답시간 p95 ≤ 300ms 달성 (소켓 유지 비용이 들지만 리소스 누수가 없어야 함 `asyncio.sleep` 활용)
- 안정성: 에러율 ≤ 0.5% 유지, 클라이언트가 탭을 닫아 `ClientDisconnect` 발생 시 즉시 백엔드루프 종료 및 메모리 반환
- 보안: 비밀번호 평문 저장 절대 금지, 요청 페이로드 로깅 시 마스킹 처리 필수

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 단위 테스트(Unit Test) 및 통합 테스트(Integration Test)가 추가되었고 통과하는가?
- [ ] SonarQube / Linter 등의 정적 분석 도구 경고가 없는가?
- [ ] API 명세서(Swagger 등)가 최신화되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-002` (Redis 인프라)
- Blocks: `VIEW-016` (프론트 통신 레이어)
