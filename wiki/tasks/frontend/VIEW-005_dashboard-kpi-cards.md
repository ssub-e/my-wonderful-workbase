---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-005: 대시보드 최상단 KPI 위젯 카드 구현"
labels: 'frontend, view, dashboard, phase:5, priority:critical'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-005] 대시보드 화면용 요약 KPI 타일(Summary Card) 배치
- 목적: `DASH-001` 백엔드 API에서 던져주는 데이터(전일 실제 주문량, 내일 AI 예측량, AI 신뢰도 평균)를 3~4개의 이쁜 사각형 카드 타일에 맵핑하여 화면 상단에 일렬로 렌더링한다. 증감율에 따른 색상(녹색/적색) 처리 로직이 돋보인다.

## 🔗 References (Spec & Context)
- 연결 API: `DASH-001` (GET `/dashboard/summary`)
- 디자인 토큰: `VIEW-002` 내 `Card` 컴포넌트 조합 활용.

## ✅ Task Breakdown (실행 계획)

### 1. Data Fetching Hook 작성
- [ ] `src/hooks/useDashboardSummary.ts`:
  ```typescript
  import { useQuery } from '@tanstack/react-query';
  import { apiClient } from '@/api/client';

  export const useDashboardSummary = () => {
    return useQuery({
      queryKey: ['dashboard', 'summary'],
      queryFn: async () => {
        const { data } = await apiClient.get('/dashboard/summary');
        return data; // DASH-001 의 DashboardKPISummary 객체 반환
      },
      staleTime: 1000 * 60 * 5, // 5분 캐싱
    });
  };
  ```

### 2. UI 컴포넌트 구성 (단일 타일)
- [ ] `src/components/dashboard/StatCard.tsx`:
  - `title` (예: 내일 출고 예측량), `value` (예: 5,124 건), `variance` (예: +12%) 를 Props로 받는다.
  - `variance` 가 양수면 `text-success` (녹색 화살표), 음수면 `text-danger` (적색 화살표) 로 조건부 렌더링한다.

### 3. 상단 Grid 병합 본체
- [ ] `src/pages/DashboardPage.tsx` 스켈레톤:
  ```tsx
  import { useDashboardSummary } from '@/hooks/useDashboardSummary';
  import { StatCard } from '@/components/dashboard/StatCard';
  
  export const DashboardPage = () => {
    const { data, isLoading, isError } = useDashboardSummary();

    if (isLoading) return <div>Data Loading Spinner...</div>;

    return (
      <div className="space-y-6">
        <h1 className="text-2xl font-bold text-gray-900">물류 예측 대시보드 종합</h1>
        
        {/* KPI 통계 카드 레이아웃 (Grid) */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
          <StatCard 
             title="전일 실제 출고량" 
             value={`${data.yesterday_actual_qty.toLocaleString()} Box`} 
          />
          <StatCard 
             title="익일 예측 출고량" 
             value={`${data.tomorrow_predicted_qty.toLocaleString()} Box`}
             variance={data.qty_variance_percent} 
          />
          <StatCard 
             title="AI 예측 신뢰도 평균" 
             value={`${(data.avg_confidence_level * 100).toFixed(1)}%`} 
          />
        </div>
        
        {/* 추후 여기에 차트 컴포넌트 위치 */}
      </div>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 상태 증감율 UI 피드백 컬러링**
- Given: 백엔드에서 `qty_variance_percent: -4.5` (감소) 값이 날아옴.
- When: 화면이 로드되어 렌더링 돔 판단이 들어감.
- Then: `StatCard` 내부는 해당 값이 음수임을 판별하여 ↓ 아이콘과 함께 빨간색 텍스트(ex: Tailwind `text-danger`)로 값을 입혀주어 담당자가 직관적으로 물량 감소 추세를 파악케 한다 (마이너스 수치임에 기한).

## ⚙️ Technical Constraints
- 데이터 포매팅: 출고량 같은 1,000 단위를 넘는 숫자는 무조건 `.toLocaleString()` 등 Utils 를 거쳐 천 단위 콤마형 서식을 뿌려주어야 엔터프라이즈 레벨의 가독성을 갖출 수 있다.

## 🏁 Definition of Done (DoD)
- [ ] React-Query Hook 연동 코드가 작성되었는가?
- [ ] 수치 부호 분기에 따른 색상 조건부 렌더링 클래스가 존재?
- [ ] Grid 를 활용한 3-Column 반응형 레이아웃 배치 완료?
