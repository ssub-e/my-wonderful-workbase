---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-021: 대시보드 사이드바 '협업 메모 피드(Feed)' 및 타임라인 UI"
labels: 'feature, frontend, ux, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-021] 대시보드 실시간 협업 메모 사이드 패널
- 목적: REQ-FUNC-010 확장. 대시보드 우측에 슬라이딩 형태의 사이드바를 배치하여, 현재 보고 있는 차트의 날짜와 연동된 메모 스레드를 보여준다. 유저들은 여기서 실시간으로 의견을 교환하거나 과거 MD들이 남긴 주석을 읽으며 의사결정 근거를 확인한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 백엔드 의존성: `NOTE-001` (Annotation API)
- 디자인 시스템: `VIEW-002` (Tailwind 공통 UI)

## ✅ Task Breakdown (실행 계획)
- [ ] `SideSheet` 또는 `Drawer` 형태의 레이아웃 컴포넌트 개발 (Tailwind Transition 활용)
- [ ] 현재 선택된 데이터 포인트(`VIEW-019` 연계) 혹은 전체 타임라인 전환 토글 구현
- [ ] 메모 목록 렌더링: 유저 아바타, 이름, 작성 시간, 내용, 답글 버튼
- [ ] `NOTE-001` API를 활용한 실시간 메모 작성 폼 (`react-hook-form` 연계)
- [ ] 낙관적 업데이트(Optimistic UI) 적용: 메모 전송 즉시 리스트에 추가하고 서버 응답 대기

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 메모 사이드 패널 열기/닫기
- Given: 대시보드 메인 화면
- When: 우측 상단의 '메모(Chat)' 아이콘을 클릭함
- Then: 우측에서 부드럽게 사이드바가 나타나며, 최신 협업 피드가 날짜순으로 로드된다.

Scenario 2: 특정 예측 포인트 선택 시 메모 필터링
- Given: 12월 25일 예측 포인트를 클릭하여 드릴다운 모달(`VIEW-019`)이 열린 상태
- When: 모달 내 '이 날짜의 메모 보기' 버튼 클릭
- Then: 사이드바가 열리면서 12월 25일에 작성된 주석들만 필터링되어 강조 표시된다.

## ⚙️ Technical & Non-Functional Constraints
- UX: 모바일 뷰 대응 (Mobile에서는 Full Screen Overlay로 전환)
- 성능: 메모가 많아질 경우 가상 스크롤(Virtual Scrolling) 적용 검토
- 접근성: `Aria-hidden` 속성 관리 및 키보드 `Esc` 키로 닫기 지원

## 💻 Implementation Snippet (React Component Idea)
```tsx
const MemoSidebar = ({ forecastId, isOpen, onClose }: Props) => {
  const { data: memos, isLoading } = useQuery(['memos', forecastId], () => fetchMemos(forecastId));
  const mutation = useMutation(postMemo, {
    onMutate: async (newMemo) => {
      // Optimistic Update logic here
    }
  });

  return (
    <div className={cn("fixed right-0 top-0 h-full w-80 bg-white shadow-xl transition-transform", isOpen ? "translate-x-0" : "translate-x-full")}>
      <div className="p-4 border-b flex justify-between items-center">
        <h3 className="font-bold">Collaboration Feed</h3>
        <button onClick={onClose}><XIcon /></button>
      </div>
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
         {memos?.map(memo => <MemoCard key={memo.id} memo={memo} />)}
      </div>
      <MemoInputForm onSubmit={(val) => mutation.mutate(val)} />
    </div>
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 메모 작성 및 답글 달기 기능이 백엔드 API와 연동 완료되었는가?
- [ ] 장문의 메모 입력 시 말줄임표(...) 및 '더보기' 처리가 적절한가?

## 🚧 Dependencies & Blockers
- Depends on: `NOTE-001`, `VIEW-004` (레이아웃)
- Blocks: 없음
