---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-023: 신규 사용자를 위한 '인터랙티브 온보딩 가이드 투어' UI"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-023] 대시보드 기능 안내를 위한 단계별 오버레이 가이드
- 목적: B2B SaaS의 문턱(Learning Curve)을 낮춘다. 처음 가입한 유저가 대시보드에 진입했을 때, "어디서 예측 결과를 보는지", "어떻게 메모를 남기는지", "보고서는 어디서 뽑는지"를 실제 화면 위에서 강조(Highlight)하며 단계별로 가이드하는 튜토리얼을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 라이브러리 선정: `Intro.js` 또는 `React Joyride`
- 타겟 요소: `VIEW-005`(StatCard), `VIEW-006`(Chart), `VIEW-019`(Drill-down)

## ✅ Task Breakdown (실행 계획)
- [ ] 온보딩 투어 설정 객체(`steps.ts`) 정의: 각 스텝별 CSS 셀렉터, 제목, 가이드 텍스트 매핑
- [ ] 처음 로그인한 유저인지 체크하는 전역 상태(`is_first_login`) 연동
- [ ] **Step 1: 메인 KPI 카드**: "한 눈에 오늘 물동량을 파악하세요" 강조
- [ ] **Step 2: 시계열 차트**: "미래 예측 점선을 확인하는 방법" 안내
- [ ] **Step 3: 사이드바 협업**: "동료와 소통하는 법" 강조
- [ ] 사용자가 직접 '나중에 하기' 또는 '다신 보지 않기' 클릭 시 DB(`USER_PREFERENCE`)에 영구 저장

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 신규 유저 최초 진입 시 자동 시작
- Given: 가입 후 처음 로그인한 MD
- When: 대시보드 엔트리 포인트 진입
- Then: 전체 화면이 흐려지며(보조 오버레이), 첫 번째 KPI 카드에 밝은 스포트라이트가 켜지고 안내 팝업이 등장한다.

Scenario 2: 온보딩 도중 이탈 및 재개 방지
- Given: 온보딩 가이드가 진행 중인 상태
- When: 유저가 '모두 건너뛰기'를 클릭함
- Then: 오버레이가 즉시 사라지며, 해당 유저는 다시 접속해도 온보딩 가이드가 자동으로 뜨지 않아야 한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: '이전/다음' 버튼 및 현재 진행률(예: 2/5) 표시 필수
- 반응형: 모바일 기기에서는 하이라이트 위치가 어긋나지 않도록 뷰포트 계산 로직 적용
- 성능: 온보딩 라이브러리 번들 사이즈가 크므로 코드 스플리팅(`Lazy Loading`) 적용

## 💻 Implementation Snippet (React Joyride Integration)
```tsx
const ONBOARDING_STEPS = [
  {
    target: '#kpi-grid',
    content: '여기서 전사 물동량 요약을 한 눈에 확인하세요.',
  },
  {
    target: '.recharts-surface',
    content: '점선으로 표시된 부분은 AI가 예측한 미래의 값입니다.',
  },
  {
    target: '#memo-toggle',
    content: '특이 사항이 있다면 메모를 남겨 동료와 공유하세요.',
  }
];

export const OnboardingTour = () => {
  const { user, setPreference } = useAuth();
  
  if (!user.showOnboarding) return null;

  return (
    <Joyride
      steps={ONBOARDING_STEPS}
      continuous={true}
      showProgress={true}
      callback={(data) => {
          if (data.status === 'finished') setPreference({ showOnboarding: false });
      }}
    />
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 가이드 텍스트가 다국어(`I18N-001`) JSON에 모두 등록되었는가?
- [ ] 투어 종료 시 백엔드 API를 통해 유저의 '온보딩 완료' 플래그가 정상 업데이트되는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-001`, `VIEW-004`, `VIEW-022` (Preferences)
- Blocks: 없음
