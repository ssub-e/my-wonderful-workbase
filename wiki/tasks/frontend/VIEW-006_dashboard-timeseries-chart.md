---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-006: 시계열 예측/실적 병합 차트 (Recharts) 연동"
labels: 'frontend, view, dashboard, chart, phase:5, priority:high'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-006] 대시보드 하단 - 메인 실시간 시계열(Time-series) 차트 렌더링
- 목적: `DASH-002`에서 건네받은 과거 14일 / 미래 7일의 Array 데이터를 `Recharts` 라이브러리를 통해 이쁜 콤보 그래프(실선+점선 혼합)로 그린다. 과거는 실제 주문량 라인, 미래는 예측 출고량 점선 라인으로 표현하여 XAI 시각화의 토대를 제공한다.

## 🔗 References (Spec & Context)
- 연결 API: `DASH-002` (GET `/dashboard/chart`)
- 차트 렌더러 스택: C-TEC-010 (Recharts 라이브러리 사용 권장 - 번들 사이즈 최적화 및 React 호환성).

## ✅ Task Breakdown (실행 계획)

### 1. Data Fetching Hook 
- [ ] `src/hooks/useTimeseriesChart.ts` 에서 `React-Query` 로 데이터 Fetch. (단, 컴포넌트 렌더링 지연을 막기 위해 Loading State 스켈레톤 관리 필수).

### 2. Recharts Area/Line 복합 차트 렌더링 컴포넌트
- [ ] `src/components/dashboard/OrderTrendChart.tsx`:
  ```tsx
  import { ResponsiveContainer, ComposedChart, Line, Area, XAxis, YAxis, CartesianGrid, Tooltip, Legend } from 'recharts';

  export const OrderTrendChart = ({ data }: { data: any[] }) => {
    // data = [{ date: "05/01", actual_qty: 400, predicted_qty: null }, ... 미래 데이터]
    return (
      <div className="h-96 w-full p-4 bg-white rounded-xl shadow-sm border">
        <h2 className="text-lg font-semibold mb-4">과거 출고량 및 AI 7일 단기 예측 지표</h2>
        <ResponsiveContainer width="100%" height="100%">
          <ComposedChart data={data} margin={{ top: 10, right: 30, left: 0, bottom: 0 }}>
            {/* 배경 격자선과 X/Y 축 */}
            <CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#E5E7EB" />
            <XAxis dataKey="date" tick={{fontSize: 12}} />
            <YAxis tickFormatter={(val) => `${val.toLocaleString()}`} />
            
            {/* 호버 동작 시 정보창 */}
            <Tooltip formatter={(value: number) => [`${value} Box`, "수량"]} />
            <Legend />
            
            {/* 과거 실제 출고량 (Area로 채워진 실선 영역) */}
            <Area 
               type="monotone" 
               dataKey="actual_qty" 
               name="실제 출고" 
               stroke="#1E3A8A" 
               fill="#DBEAFE" 
            />
            
            {/* 미래 예측 출고량 (점선 Line 처리) */}
            <Line 
               type="monotone" 
               dataKey="predicted_qty" 
               name="AI 예측 (신뢰도 85%)" 
               stroke="#10B981" 
               strokeWidth={2}
               strokeDasharray="5 5" // 점선 처리
               dot={{ r: 4 }} 
            />
          </ComposedChart>
        </ResponsiveContainer>
      </div>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: '오늘' 날짜를 넘나드는 선 겹침 랜더링 무결성**
- Given: 과거에서 미래로 넘어가는 과도기 노드 1개가 `actual_qty: 1560`, `predicted_qty: 1560` 둘 다 보유함.
- When: Recharts 엔진이 위 레이아웃 지시대로 그려냄.
- Then: 파란색 `Area` 실선이 쭉 오다가 오늘 날짜 포인트에서 딱 끊어지고, 같은 시작 포인트에서부터 녹색 점선 `Line` 이 뻗어 나감으로써 과거와 미래 예측이 아름답게 조화되는 시계열 콤보차트가 시연된다. 

**Scenario 2: Y축 동적 스케일링**
- Given: 주문량이 평소 100건대이던 샵에 인플루언서 붐으로 갑자기 20,000건 폭등 데이터가 찍힘.
- When: 브라우저 Resize나 로딩이 진행됨.
- Then: `Recharts` 자체 YAxis 의 오토 스케일링 혹은 `Tooltip` 단위가 컴포넌트 밖의 부모 영역(Div) 박스를 뚫고 나가는 등 프레임 훼손 에러를 발생시키지 않고 차분히 자동 리스케일링 된다 (`ResponsiveContainer width=100%`).

## ⚙️ Technical Constraints
- 라이브러리: 차트 라이브러리(`Recharts`)가 번들 사이즈를 꽤 차지할 수 있으므로, 해당 대시보드 컴포넌트는 추후 최적화 단계에서 `React.lazy()` 로 Code-splitting(코드 분할)되어 지연 로딩됨을 염두에 두고 독립화시킨다.

## 🏁 Definition of Done (DoD)
- [ ] `Area(과거)` 와 `Line(미래 점선)` 컴포넌트 듀얼 매핑 구문 포함?
- [ ] 천단위 콤마 포맷터(Tooltip, YAxis)가 적용되어 있는가?
- [ ] `ResponsiveContainer` 로 랩핑하여 브라우저 확대/축소에 무너지지 않는지 통제 룰 적용?
