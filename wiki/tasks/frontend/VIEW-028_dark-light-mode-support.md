---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-028: 사용자 맞춤형 '다크/라이트 모드(Theme)' 및 브랜드 컬러 영구 적용 UI"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-028] 시스템 전반 테마 전환 및 사용자별 영구 테마 저장
- 목적: B2B SaaS의 사용 편의성을 극대화한다. 장시간 대시보드를 모니터링하는 MD의 눈의 피로를 덜어주기 위한 '다크 모드'를 지원하고, `ADMIN-009` 에서 설정한 브랜드 컬러가 라이트/다크 양환경에서 모두 조화롭게 렌더링되도록 시각 체계를 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- UI 프레임워크: `Tailwind CSS (dark mode: class)`
- 브랜드 연동: `ADMIN-009` (Tenant Branding)
- 관련 태스크: `VIEW-022` (User Preferences)

## ✅ Task Breakdown (실행 계획)
- [ ] **Theme Context Provider** 개발: 시스템 설정, 사용자 설정, 브라우저 환경 설정(`prefers-color-scheme`) 간 우선순위 정의
- [ ] **Tailwind Config 확장**: 다크 모드용 컬러 팔레트 정의 (Slate/Zinc 계열의 가독성 높은 배경색 선정)
- [ ] **CSS Variables 바인딩**: 브랜드 컬러(Primary)가 다크 모드에서도 충분한 명암비(Contrast Ratio)를 갖도록 자동 보정 로직 적용
- [ ] **Toggle UI Implementation**: 헤더 또는 프로필 설정 창에 부드러운 애니메이션이 적용된 '해/달' 전환 스위치 배치
- [ ] **State Persistence**: 유저가 선택한 테마를 `Local Storage` 에 즉시 저장하고, 로그인 상태인 경우 서버 비동기 저장(`VIEW-022`)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 다크 모드 수동 전환 및 유지
- Given: 대시보드 화면 하단
- When: 테마 스위치를 'Dark'로 클릭함
- Then: HTML 태그에 `dark` 클래스가 주입되며 즉시 전체 배경색이 어두운 톤으로 변경되고, 새로고침 후에도 유지되어야 한다.

Scenario 2: 브랜드 컬러 명암비 자동 보정
- Given: 테넌트 브랜드 컬러가 아주 연한 노란색(라이트 전용)임
- When: 사용자가 다크 모드로 전환함
- Then: 가독성을 위해 해당 브랜드 컬러의 채도/명도를 다크 모드용으로 미세 조정하여 텍스트 가독성을 확보한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: 전환 시 갑작스러운 '깜빡임(Flash of Unstyled Content)' 방지를 위해 `next-themes` 패턴의 초기 인라인 스크립트 적용
- 디자인: 모든 차트(`Recharts`)의 색상 테마도 다크 모드 감지 시 가독성 높은 네온/고명도 색상으로 자동 변경
- 가용성: 저사양 모니터에서도 그리드 선이 잘 보이도록 명암 대비 최적화

## 💻 Implementation Snippet (Theme Logic Idea)
```tsx
const useTheme = () => {
    const [theme, setTheme] = useState(localStorage.getItem('theme') || 'light');

    useEffect(() => {
        const root = window.document.documentElement;
        root.classList.remove('light', 'dark');
        root.classList.add(theme);
        localStorage.setItem('theme', theme);
    }, [theme]);

    return { theme, toggle: () => setTheme(prev => prev === 'light' ? 'dark' : 'light') };
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 모든 페이지(대시보드, 설정, 리포트 목록 등)에서 다크 모드 누락 요소가 없는가?
- [ ] 모바일 환경(`VIEW-026`)에서도 테마 전환 UI가 적절히 노출되는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-001`, `ADMIN-009`
- Blocks: 없음
