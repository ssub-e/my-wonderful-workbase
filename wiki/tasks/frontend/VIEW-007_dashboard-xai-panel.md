---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-007: 대시보드 XAI(설명 가능한 AI) 해설 패널 컴포넌트"
labels: 'frontend, view, dashboard, xai, phase:5, priority:high'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-007] 왜(Why) 예측이 그렇게 도출되었는지를 보여주는 Fact-Sheet 패널
- 목적: B2B 고객의 물류센터장이 대시보드 차트(`VIEW-006`)의 점선을 눌렀을 때, 우측이나 하단에 해당 일자의 예측 근거(SHAP 요인 그래프)와 LLM(Gemini)이 번역한 한글 해설을 렌더링하는 컴포넌트를 제공한다.

## 🔗 References (Spec & Context)
- 연결 API: `DASH-003` (GET `/xai/{sku}/explanation`)
- 연관 UI: `VIEW-002` (디자인 시스템 Card 사용)

## ✅ Task Breakdown (실행 계획)

### 1. Data Fetching Hook 
- [ ] `src/hooks/useXAIExplanation.ts`:
  - `DASH-003` 엔드포인트를 때려 `key_factors` (요인 리스트)와 `human_readable_explanation` (자연어 텍스트)를 가져오는 React Query 훅 작성.

### 2. UI 컴포넌트 렌더링
- [ ] `src/components/dashboard/XAIPanel.tsx`:
  ```tsx
  import { useXAIExplanation } from '@/hooks/useXAIExplanation';
  import { Card } from '@/components/ui/Card';
  import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, Cell } from 'recharts';

  export const XAIPanel = ({ sku, targetDate }: { sku: string, targetDate: string }) => {
    const { data, isLoading } = useXAIExplanation(sku, targetDate);

    if (isLoading) return <Card className="p-6">분석 결과를 불러오는 중입니다...</Card>;
    if (!data) return <Card className="p-6">XAI 데이터가 없습니다.</Card>;

    return (
      <Card className="p-6 flex flex-col gap-4">
        <h3 className="text-lg font-bold">💡 AI 예측 분석 리포트 ({targetDate})</h3>
        
        {/* LLM 요약 텍스트 구역 */}
        <div className="bg-blue-50 p-4 rounded-lg border border-blue-100 text-sm text-gray-800 leading-relaxed">
          {data.human_readable_explanation}
        </div>

        {/* SHAP 요인 가로 막대 그래프 */}
        <div className="h-64 mt-4 relative">
          <h4 className="text-sm font-semibold mb-2 text-gray-500">주요 영향 요인 (SHAP)</h4>
          <ResponsiveContainer width="100%" height="100%">
            <BarChart layout="vertical" data={data.key_factors} margin={{ left: 40 }}>
              <XAxis type="number" />
              <YAxis dataKey="factor_name" type="category" width={80} tick={{fontSize: 12}} />
              <Tooltip />
              <Bar dataKey="impact_value" radius={[0, 4, 4, 0]}>
                {data.key_factors.map((entry, index) => (
                  <Cell key={`cell-${index}`} fill={entry.impact_value > 0 ? '#10B981' : '#EF4444'} />
                ))}
              </Bar>
            </BarChart>
          </ResponsiveContainer>
        </div>
      </Card>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: SHAP +/- 부호에 따른 색상 분기 렌더링**
- Given: 백엔드에서 강수량 요인(`impact_value: -12.5`)과 요일 요인(`impact_value: +8.0`)이 날아옴.
- When: `Recharts` 의 `BarChart` 가 렌더링됨.
- Then: 배열 안에서 `.map()` 을 돌며 `impact_value > 0` 여부를 판단하여, +8.0인 막대는 녹색(`success`)으로, -12.5인 막대는 적색(`danger`)으로 직관적으로 칠해져 센터장의 판단을 돕는다.

## ⚙️ Technical Constraints
- X축(수치 축)이 음수를 지원해야 하므로, `Recharts` 에서 커스텀 도메인 이나 기본 음수 차트 처리가 어긋나지 않는지 점검한다.
- 텍스트 길이가 꽤 길 수 있으므로 `leading-relaxed` 등으로 B2B 가독성을 높인다.

## 🏁 Definition of Done (DoD)
- [ ] LLM 자연어를 담는 Alert/Box 디자인이 가독성이 있는가?
- [ ] 양/음수에 반응하여 녹색/적색으로 칠해지는 가로 막대 그래프 코드가 작성?
