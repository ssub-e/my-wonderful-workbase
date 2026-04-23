---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-026: 현장 MD를 위한 '모바일 가로 맞춤(Responsive) 사이드바 및 네비게이션' 최적화"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-026] 모바일 우선(Mobile-first) 대시보드 반응형 레이아웃 강화
- 목적: REQ-NF-004 보장. 물류 현장 관리자가 창고에서 모바일 기기나 태블릿을 들고 다니며 실시간 수치를 확인할 수 있도록, 기존 데스크톱 위주의 사이드바를 '모바일 햄버거 메뉴'와 '하단 고정 탭 바' 형식으로 자동 전환되는 반응형 내비게이션 구조로 고도화한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 기존 레이아웃: `VIEW-004` (GNB/LNB)
- 디자인 가이드: Tailwind CSS 브레이크포인트 (`md:`, `sm:`) 활용

## ✅ Task Breakdown (실행 계획)
- [ ] **Responsive Sidebar Container**: 데스크톱에서는 슬라이딩 사이드바, 모바일에서는 좌측 오버레이(Sheet) 메뉴로 전환
- [ ] **Mobile Hamburger Button**: 상단 헤더 좌측에 메뉴 열기/닫기 인터랙션 구현 (`Drawer` 컴포넌트)
- [ ] **Bottom Navigation Bar (Mobile Only)**: 가장 자주 쓰는 메뉴 4개(홈, 메모, 리포트, 알림)를 화면 하단에 고정 배치
- [ ] **Touch Target Optimization**: 모든 클릭 요소의 크기를 최소 44x44px 이상으로 조정하여 오클릭 방지
- [ ] **Chart Resize Logic**: 모바일 세로 모드에서 차트의 범례(Legend)를 아래로 내리거나 툴팁을 터치 최적화로 변경

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 화면 너비에 따른 레이아웃 전환
- Given: 리사이즈 가능한 브라우저 창
- When: 너비를 768px(md) 이하로 줄임
- Then: 좌측 고정 사이드바가 사라지고 대신 상단 햄버거 메뉴 버튼과 하단 탭 바가 나타난다.

Scenario 2: 모바일 메뉴 이탈 및 복귀 제어
- Given: 모바일 메뉴(Sheet)가 열려 있는 상태
- When: 메뉴 항목 중 하나를 클릭하거나 외부 영역을 터치함
- Then: 메뉴가 부드럽게 닫히며 해당 페이지로 이동하거나 원래 화면이 보여야 한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: 모바일에서 차트 스크롤 시 페이지 전체 스크롤과 간섭되지 않도록 `Touch-Action: pan-y` 속성 조율
- 성능: 모바일 저사양 기기에서의 애니메이션 렉(Lag) 방지를 위해 `Framer Motion` 의 `layout` 최적화 사용
- 가용성: 오프라인 상황(현장 음영 지역)에서도 레이아웃은 유지되도록 스켈레톤 UI 기본 적용

## 💻 Implementation Snippet (Responsive Menu Switcher)
```tsx
const RootLayout = ({ children }: { children: React.ReactNode }) => {
  const isMobile = useMediaQuery("(max-width: 768px)");

  return (
    <div className="flex min-h-screen bg-gray-50">
      {/* Desktop Sidebar */}
      {!isMobile && <Sidebar className="w-64 border-r" />}
      
      <div className="flex-1 flex flex-col">
        <header className="h-16 border-b flex items-center px-4">
          {isMobile && <MobileDrawerMenu />}
          <UserNav />
        </header>

        <main className="flex-1 p-4 pb-20 md:pb-4">{children}</main>
        
        {/* Mobile Bottom Bar */}
        {isMobile && <BottomNav className="fixed bottom-0 w-full" />}
      </div>
    </div>
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] iPhone, Android, iPad 등 주요 모바일 뷰포트에서 레이아웃 깨짐이 없는가?
- [ ] 하단 탭 바 클릭 시 React Router의 액티브 링크 상태가 정확히 표시되는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-004`
- Blocks: 없음
