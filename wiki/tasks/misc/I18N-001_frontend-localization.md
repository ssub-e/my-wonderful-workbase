---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] I18N-001: react-i18next 기반 다국어 지원 아키텍처 및 번역 리소스"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [I18N-001] 글로벌 시장 대응을 위한 다국어(KOR/ENG) 전환 인프라
- 목적: B2B 시스템이 국내를 넘어 글로벌 화주사나 영어권 물류 담당자에게도 서비스될 수 있도록 모든 하드코딩된 문자열을 번역 키(Key)로 치환하고, 유저의 Locale 설정에 따라 즉각적인 언어 전환 기능을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 가이드라인: `react-i18next` 공식 문서
- 대상 파일: `src/locales/ko/common.json`, `src/locales/en/common.json`

## ✅ Task Breakdown (실행 계획)
- [ ] `i18next`, `react-i18next`, `i18next-browser-languagedetector` 라이브러리 설치
- [ ] `src/i18n.ts`: 초기화 설정 및 기본 언어(ko-KR) 세팅
- [ ] 도메인별(Dashboard, Settings, Auth) 번역 JSON 파일 구조화 및 공통 Key 추출
- [ ] 프론트엔드 UI 컴포넌트 내 `useTranslation` 훅을 사용해 평문을 `t('key')` 로 교체
- [ ] 우측 상단 유저 설정 메뉴 내 '언어 선택(Language Switcher)' 전용 드롭다운 UI 추가

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 브라우저 기본 언어 자동 감지
- Given: 브라우저 언어 설정이 'English' 인 신규 유저가 접속
- When: 대시보드 페이지가 처음 로드됨
- Then: 로그인 화면의 레이블이 "로그인" 이 아닌 "Login" 으로, 버튼 텍스트가 "Enter" 로 자동 렌더링된다.

Scenario 2: 실시간 언어 스위칭 (Dynamic Switch)
- Given: 현재 한국어 대시보드 화면을 보고 있는 중
- When: 언어 스위처 드롭다운에서 'English' 를 선택함
- Then: 페이지 새로고침 없이 즉시 "예측 출하량" 이 "Predicted Shipment" 로, "권장 인원" 이 "Recommended Workers" 로 실시간 변환된다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 번역 파일 용량 최적화 (Lazy Loading 기법을 적용하여 초기 번들 크기 증가 억제)
- 안정성: 누락된 번역 키(Missing Key) 발생 시 Fallback으로 원시 키값 혹은 한국어 텍스트 노출 보장
- 보안: 사용자 locale 정보를 `localStorage` 와 유저 프로필 DB에 동시 저장하여 로그인 시 상태 복원

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 한국어/영어 번역 JSON 파일이 각각 최소 50개 이상의 핵심 UI 키를 포함하는가?
- [ ] CI 파이프라인에서 번역 파일 누락 여부를 스캔하는 간단한 린트 체크 추가 여부?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-001`, `VIEW-004` (레이아웃 구성 선행)
- Blocks: `I18N-002` (백엔드 로케일 연동)
