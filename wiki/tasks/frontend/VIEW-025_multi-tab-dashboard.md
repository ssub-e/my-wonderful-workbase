---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-025: 분석 생산성 향상을 위한 '멀티 탭(Multi-tab) 대시보드' 인터페이스"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-025] 여러 분석 뷰를 동시 탐색하는 대시보드 탭 시스템
- 목적: 고도화된 MD 환경에서는 SKU A의 시계열과 SKU B의 재고 현황을 번갈아 보며 비교해야 하는 상황이 잦다. 브라우저 탭을 여러 개 띄우는 불편함을 없애기 위해, 대시보드 내부에 '내부 탭' 시스템을 구축하여 여러 분석 컨텍스트를 유지한 채 자유롭게 전환하며 작업할 수 있게 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 프레임워크: `Zustand` 또는 `Redux` (탭 상태 관리)
- 관련 태스크: `VIEW-004` (레이아웃), `VIEW-019` (드릴다운)

## ✅ Task Breakdown (실행 계획)
- [ ] **Tab Manager** 전역 상태 개발: `activeTabs` 배열(id, title, path, filters) 관리
- [ ] 대시보드 상단 탭 바 UI 구현: `shadcn/ui`의 Tabs 또는 커스텀 슬라이딩 탭
- [ ] **Context Preservation**: 탭을 전환할 때 이전에 설정했던 필터값(`VIEW-024`)이나 차트 확대 수준이 유지되도록 `SessionStorage` 연동
- [ ] '탭 닫기', '현재 탭 외 모두 닫기', '탭 순서 드래그 앤 드롭' 기능 구현
- [ ] **Keep-Alive 렌더링**: 비활성 탭이 배경에서 완전히 언마운트되지 않고 데이터만 캐싱되어 전환 시 즉시 나타나도록 최적화
- [ ] 탭이 너무 많아질 경우 가로 스크롤 처리 및 검색 브라우징 기능

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 다중 분석 뷰 생성 및 전환
- Given: 대시보드 메인 화면
- When: 상품 A 조회 중 '새 탭에서 열기' 클릭 후 상품 B 선택
- Then: 상단에 2개의 탭이 생기며, 상품 A 탭으로 돌아왔을 때 이전에 보고 있던 차트 지점이 그대로 유지되어야 한다.

Scenario 2: 최대 탭 개수 제한 및 성능
- Given: 10개의 분석 탭이 열린 상태
- When: 11번째 탭을 열려고 시도함
- Then: "최적의 성능을 위해 최대 10개까지 열 수 있습니다." 라는 알림과 함께 가장 오래된 탭을 닫거나 안내함.

## ⚙️ Technical & Non-Functional Constraints
- UX: 탭 개별 로딩 스피너 제공
- 성능: 탭 전환 시 중복 API 요청 방지를 위한 `staleTime` (React Query) 조정
- 가용성: 브라우저 강제 종료 후 재접속 시 이전 탭 목록 복구 기능 (User Preference 연동)

## 💻 Implementation Snippet (Zustand Tab Store)
```typescript
interface DashboardTab {
  id: string;
  title: string;
  path: string;
  filters: object;
}

const useTabStore = create((set) => ({
  tabs: [],
  activeTabId: null,
  addTab: (tab: DashboardTab) => set((state) => ({ 
      tabs: [...state.tabs, tab],
      activeTabId: tab.id 
  })),
  removeTab: (id: string) => set((state) => ({
      tabs: state.tabs.filter(t => t.id !== id),
      activeTabId: state.tabs[0]?.id || null
  }))
}));
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 탭 간의 필터 값이 서로 간섭(Leakage)하지 않는가?
- [ ] 모바일 환경에서는 탭 바가 숨겨지고 '탭 리스트' 모달로 대체되는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-004`, `VIEW-024`
- Blocks: 없음
