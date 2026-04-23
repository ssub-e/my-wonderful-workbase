---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] I18N-002: 백엔드 다국어 에러 메시지 및 리포트 Locale 처리"
labels: 'feature, backend, priority:low'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [I18N-002] API 응답 및 알림/리포트의 다국어(Locale) 분기 엔진
- 목적: 프론트엔드 텍스트뿐 아니라, 서버에서 발생하는 에러 메시지(`400 Bad Request` 등), 카카오 알림톡/SMS(`F3-W03`), 그리고 생성되는 PDF 리포트(`F1-W07`)의 명칭들을 유저의 Locale 설정에 맞춰 서버 사이드에서 변환하여 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#4.1.4 공통`](#)
- 관련 기능: `F1-W07` (PDF 생성), `ADAPT-004` (알림톡)

## ✅ Task Breakdown (실행 계획)
- [ ] FastAPI `Accept-Language` 헤더 파싱 미들웨어 구현 (Default: ko)
- [ ] 백엔드용 번역 리소스 매니저 클래스 (`app/core/i18n.py`) 작성
- [ ] 에러 핸들러 클래스 개조: 고정 텍스트 대신 `gettext` 스타일의 Locale 기반 메시지 반환
- [ ] LLM 해설 생성 모듈(`F1-W06`) 호출 시 유저 Locale 정보를 프롬프트에 주입하여 영어/한국어 중 선택 출력 유도
- [ ] PDF 진자(Jinja2) 템플릿 내의 하드코딩된 레이블을 다국어 변수로 치환

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 영어 계정 유저의 PDF 리포트 생성
- Given: User Profile의 `preferred_language` 가 'en' 인 상태
- When: PDF 리포트 생성 버튼을 클릭함
- Then: 생성된 PDF의 제목이 "수요 예측 분석 리포트" 가 아닌 "Demand Forecast Analytics Report" 로 출력되며, LLM 해설 또한 영어 문장으로 작성된다.

Scenario 2: API 에러 메시지 다국어화
- Given: 헤더에 `Accept-Language: en-US` 가 포함된 요청
- When: 중복된 이메일로 가입을 시도하여 409 에러가 발생함
- Then: 응답 JSON의 `detail` 필드에 "이미 사용 중인 이메일입니다" 대신 "Email already in use" 라는 영문 메시지가 수신된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 런타임 번역 룩업 성능 최적화 (캐싱 적용, p95 ≤ 50ms 오버헤드 제한)
- 일관성: 프론트엔드(`I18N-001`)와 백엔드 간의 번역 키 매핑 파일(JSON)을 공유하거나 싱크를 맞추는 프로세스 수립
- 안정성: 지원하지 않는 Locale 요청 시 기본 언어(ko)로 Fallback

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 주요 API 에러 메시지 20종 이상에 대한 영문 번역이 완료되었는가?
- [ ] PDF 템플릿(`jinja2`) 변수가 다국어 대응 가능하도록 리팩토링 되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `I18N-001` (전략 공유), `F1-W07` (PDF), `F1-W06` (LLM)
- Blocks: 없음
