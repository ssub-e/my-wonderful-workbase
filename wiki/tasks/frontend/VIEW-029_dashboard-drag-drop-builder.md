---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-029: 사용자 커스텀 리포트 생성을 위한 '드래그 앤 드롭(Drag-and-Drop) 대시보드 빌더'"
labels: 'feature, frontend, ux, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-029] 나만의 분석 뷰를 구성하는 인터랙티브 대시보드 에디터
- 목적: Phase 2 고급 사양. 고정된 대시보드 레이아웃을 넘어, MD가 직접 원하는 지표 카드(`VIEW-005`)와 차트(`VIEW-006`)를 골라 빈 캔버스에 배치하고 크기를 조절하여 업무 스타일(예: 재고 중심 vs 매출 중심)에 최적화된 화면을 구성할 수 있게 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 라이브러리: `react-grid-layout` 또는 `Dnd-kit`
- 관련 태스크: `VIEW-005`~`008` (위젯 컴포넌트군), `VIEW-025` (탭 시스템 연동)

## ✅ Task Breakdown (실행 계획)
- [ ] **Widget Library Panel** 구현: 사용 가능한 모든 차트/요약 카드의 프리뷰 리스트 노출
- [ ] **Grid-based Canvas Builder**:
  - 드래그 앤 드롭으로 위젯을 캔버스 위로 배치
  - 위젯별 너비(Col) 및 높이(Row) 자유 조절 기능
- [ ] **Widget Configuration**: 개별 위젯 클릭 시 필터 조건(특정 SKU 고정 등)을 별도로 설정하는 팝오버 메뉴
- [ ] **Dashboard Metadata Schema**: 배치된 위젯들의 ID, 좌표, 크기, 설정을 JSON 형태로 정규화하여 저장
- [ ] **Preview & Publish**: 편집 모드와 조회 모드의 전환 인터랙션 구현

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 커스텀 대시보드 생성 및 저장
- Given: 대시보드 편집 모드
- When: '판매량 차트' 위젯을 캔버스로 끌어다 놓고 '저장' 클릭
- Then: 해당 유저의 개인화 설정에 레이아웃 정보가 저장되며, 다음 접속 시에도 동일한 배치가 유지된다.

Scenario 2: 위젯 크기 및 위치 조정
- Given: 기존에 배치된 3개의 위젯
- When: 우측 하단의 Resize 핸들을 잡고 2칸을 4칸으로 확장함
- Then: 나머지 위젯들이 그리드 시스템(`react-grid-layout`)에 따라 자동으로 밀려나거나 재배치되며 최적의 해상도로 렌더링된다.

## ⚙️ Technical & Non-Functional Constraints
- UX: 편집 중 '실행 취소(Undo) / 다시 실행(Redo)' 기능 제공
- 성능: 위젯이 많아질 경우 각 위젯의 데이터 로딩을 순차적(Staggered)으로 처리하여 초기 렉 방지
- 가용성: 모바일 환경(`VIEW-026`)에서는 드래그 앤 드롭 대신 '상/하 순서 변경' 버튼으로 자동 치환

## 💻 Implementation Snippet (Grid Layout Sync Idea)
```tsx
import { Responsive, WidthProvider } from "react-grid-layout";
const ResponsiveGridLayout = WidthProvider(Responsive);

const DashboardEditor = ({ layout, onLayoutChange }: Props) => {
  return (
    <ResponsiveGridLayout
      className="layout"
      layouts={{ lg: layout }}
      breakpoints={{ lg: 1200, md: 996, sm: 768 }}
      cols={{ lg: 12, md: 10, sm: 6 }}
      onLayoutChange={(currentLayout) => onLayoutChange(currentLayout)}
    >
      {layout.map((item) => (
        <div key={item.i} className="bg-white border rounded">
          <WidgetRenderer id={item.i} config={item.config} />
        </div>
      ))}
    </ResponsiveGridLayout>
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 저장된 레이아웃 JSON이 DB(`USER_PREFERENCE`)에 정확히 업로드되는가?
- [ ] 존재하지 않는 위젯 ID가 설정에 포함된 경우 에러 없이 'Missing Widget' 메시지가 노출되는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-022`, `VIEW-002`
- Blocks: 없음
