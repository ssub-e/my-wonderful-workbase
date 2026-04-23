---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-018: 대시보드 테이블 다중 필터링 및 컬럼 소팅(Sorting) 로직"
labels: 'feature, frontend, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-018] React Data Grid 고도화 (필터링 및 정렬) 
- 목적: 단순한 리스트 나열을 넘어, 물류 센터 테이블(`VIEW-008`)에서 "야간조가 필요한 센터만", "물량이 가장 많은 곳부터 내림차순(DESC) 정렬" 등 실무자 입장에서 반드시 필요한 탐색 UX를 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 모듈 컨텍스트: `VIEW-008` (Workforce Grid)
- 라이브러리 차용: `TanStack Table` (구 React Table) 등의 Headless UI 기반 활용 혹은 순수 훅(Hook) 작성

## ✅ Task Breakdown (실행 계획)
- [ ] 컬럼명(Header)을 클릭 시 `오름차순(ASC) -> 내림차순(DESC) -> 기본` 으로 순환하는 State Hook(`useSortData`) 작성
- [ ] 문자열 필터 검색바(`Search Input`) 컴포넌트 추가 및 키워드 기반 Array Filter 유틸 바인딩
- [ ] (선택) `Tanstack Table v8` 패키지 설치 및 `useReactTable` 인스턴스로 테이블 렌더 트리 전환
- [ ] 정렬 상태를 나타내는 `ChevronUp`, `ChevronDown` SVG 아이콘 헤더 내부 동적 할당
- [ ] 재사용 가능하도록 `Table` 컴포넌트 분리 (`src/components/ui/DataTable.tsx`)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 특정 수치 기준 내림차순 정렬 
- Given: 센터별 잉여 근로자 추천 예측 결과치가 섞인 채 표출됨 (A센터: 5명, B센터: 15명, C센터: 1명)
- When: 유저가 컬럼 헤더 '추천 인명' 이라는 텍스트를 클릭함
- Then: 테이블 Row가 즉시 재배열되며 [B센터(15), A센터(5), C센터(1)] 순으로 정렬되고 헤더에 역삼각형 아이콘이 뜬다.

Scenario 2: 다이나믹 키워드 필터링
- Given: 여러 센터명이 표출된 화면
- When: 상단 검색창에 "강남" 이라고 타이핑함 (onBodyChange)
- Then: 엔터(Enter)나 API 송출 없이, 프론트엔드 단의 상태 변화만으로 렌더링 리스트가 반응하여 강남과 무관한 센터 로우는 즉각 감춰진다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 응답시간 p95 ≤ 300ms 달성 (리렌더링 지연 방지를 위해 `useMemo` 를 십분 활용해 Raw Data 타겟 캐싱 필수)
- 안정성: 에러율 ≤ 0.5% 유지
- 보안: 비밀번호 평문 저장 절대 금지, 요청 페이로드 로깅 시 마스킹 처리 필수

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 단위 테스트(Unit Test) 및 통합 테스트(Integration Test)가 추가되었고 통과하는가?
- [ ] SonarQube / Linter 등의 정적 분석 도구 경고가 없는가?
- [ ] API 명세서(Swagger 등)가 최신화되었는가? (이 경우엔 Frontend Storybook 검수)

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-008`
- Blocks: 없음
