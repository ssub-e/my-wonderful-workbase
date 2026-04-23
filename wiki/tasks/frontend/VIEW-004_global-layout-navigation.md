---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-004: 어드민 글로벌 레이아웃 (LNB, GNB) 구성"
labels: 'frontend, view, layout, phase:5, priority:high'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-004] 사용자 대시보드 공통 레이아웃 (Sidebar 및 Header) 적용
- 목적: B2B 시스템 답게 좌측에는 메뉴를 네비게이션 하는 LNB(사이드바)를 배치하고 상단에는 사용자 프로필과 햄버거 버튼이 구비된 GNB(헤더)를 뼈대로 잡는다. 내부 `Outlet` 에 변하는 페이지가 렌더되는 구조다.

## 🔗 References (Spec & Context)
- 관련 API: `ADMIN-001` (상단 프로필 표출용 화주사 닉네임 Fetch)
- 아이콘: `lucide-react` (표준 B2B 아이콘 패키지 권장)

## ✅ Task Breakdown (실행 계획)

### 1. Sidebar (메뉴 패널)
- [ ] `src/components/layout/Sidebar.tsx`:
  - `lucide-react` 패키지를 사용하여 [홈(대시보드), 내일 발주 확인, 리포트 추출, 시스템 설정] 등의 메뉴를 리스트업.
  - 현재 활성화된 라우트(URL Path)와 일치하는 메뉴 항목은 바탕색을 파란색(Primary)으로 칠해 현재 위치를 인지하게 만듦 (`clsx` 동적 부여).

### 2. Header (프로필 및 유틸)
- [ ] `src/components/layout/Header.tsx`:
  - 우측 끝단에 현재 로그인 한 Tenant 이름과 User 의 직급 표시.
  - 로그아웃 드롭다운 메뉴 및 수동 동기화(`ADMIN-005` 연동) 버튼 배치.

### 3. Global Dashboard Layout 조립
- [ ] `src/components/layout/DashboardLayout.tsx`:
  ```tsx
  import { Outlet } from 'react-router-dom';
  import { Sidebar } from './Sidebar';
  import { Header } from './Header';

  export const DashboardLayout = () => {
    return (
      <div className="flex h-screen bg-background overflow-hidden">
        {/* 고정된 폭의 사이드바 */}
        <Sidebar className="w-64 border-r bg-surface" />
        
        {/* 우측 메인 뷰포트 영역 */}
        <div className="flex-1 flex flex-col min-w-0">
          <Header className="h-16 border-b shrink-0 bg-surface px-6" />
          
          <main className="flex-1 overflow-y-auto p-6">
            {/* React Router로 매핑된 개별 Page 들이 렌더링 될 공간 */}
            <Outlet />
          </main>
        </div>
      </div>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 스크롤 격리 (화면 고정 보장)**
- Given: 대시보드 내부 메인 화면 `Outlet` 영역에 매우 긴(세로로 큰) 테이블이 띄워짐.
- When: 사용자가 마우스 스크롤을 내림.
- Then: 좌측의 Sidebar 메뉴와 상단의 Header 공간은 상단에 딱 멈춰 포지션을 유지하며, 오직 메인 `<main>` 구역만 부드럽게 스크롤 바가 활성화되어 B2B 사용성의 쾌적함을 잃지 않는다. (`flex-1 overflow-y-auto` 동작 확인).

## ⚙️ Technical Constraints
- 모바일(반응형) 지원: B2B 시스템이 모바일에서 불릴 확률은 데스크톱보다 적으나, `md:` `lg:` 브레이크포인트를 적용해 모바일에서 Sidebar가 사라지고 햄버거 메뉴를 눌렀을 때 튀어나오게 하는 기초 설계(Drawer 상태값 관리) 비워두기.

## 🏁 Definition of Done (DoD)
- [ ] `Sidebar` 컴포넌트와 활성화(Active) URL 체크 로직 포함?
- [ ] `DashboardLayout` 에 Router `Outlet` 맵핑 완료?
- [ ] H-Screen 뷰포트 맞춤으로 전체 영역 스크롤 제어 적용 유무?
