---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DASH-010: 데이터 위생(Data Quality) 진단 대시보드 시각화"
labels: 'feature, frontend, ux, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-010] 데이터 신뢰도 점수(Data Health Score) 및 이상치 관리 대시보드
- 목적: REQ-FUNC-032 확장. `DQ-001` 워커가 찾아낸 시스템 오류 및 데이터 이상치(Outlier) 현황을 시각화하여, 고객사가 입력 데이터의 정합성을 스스로 진단하고 "AI 예측이 왜 이런가요?" 라는 질문에 대해 "데이터 원천이 오염되었기 때문"임을 증명하는 신뢰 도구를 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 백엔드 의존성: `DQ-001` (Anomaly Detection Logic)
- 데이터 모델: `DATA_ANOMALY` 테이블 (감지 로그)

## ✅ Task Breakdown (실행 계획)
- [ ] 대시보드 내 '데이터 관리' 탭 및 레이아웃 설계 (통계 카드 + 리스트)
- [ ] **Data Health Score (GAUGE)**: 전체 상품 중 '깨끗한' 데이터 비중을 Recharts 게이지로 표현
- [ ] **Anomaly Timeline**: 시간에 따른 이상치 감지 발생 빈도를 막대 차트로 렌더링
- [ ] **Data Grid**: 탐지된 이상치 목록 (SKU명, 감지된 값, 평균 대비 배율, 상태(Pending/Confirmed))
- [ ] **Action Button**: 특정 이상치에 대해 "이건 실제 프로모션임(무시)" 혹은 "데이터 오류임(수정)" 처리 기능 연동
- [ ] `react-query`를 이용한 실시간 건강 상태 폴링 및 뱃지 업데이트

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 데이터 건강 점수 시각화
- Given: 최근 30일간 1,000건의 수집 데이터 중 5건의 이상치가 감지됨
- When: 데이터 진단 화면에 진입함
- Then: 상단 게이지 차트가 "99.5%"(매우 우수)를 가리키며 초록색 테마로 렌더링된다.

Scenario 2: 감지된 오류에 대한 MD 확정 액션
- Given: 수지 상 오류로 감지된 리스트가 존재함
- When: MD가 특정 라인의 "정상 데이터로 승인" 버튼을 클릭함
- Then: 해당 데이터는 'Approved' 상태로 바뀌며, 즉시 예측 엔진의 피처셋(Feature Set)에 포함되도록 백엔드 상태가 변경된다.

## ⚙️ Technical & Non-Functional Constraints
- UX: 사용자가 복잡한 통계 용어(Z-score 등)를 몰라도 "데이터가 튀었다"는 것을 직관적으로 알 수 있는 도움말(Tooltip) 제공
- 성능: 수천 건의 로그 조회를 위한 인피니트 스크롤(Infinite Scroll) 적용
- 신뢰성: "데이터 오류"로 판명된 값은 차트에서 즉시 점선/X표시로 시각적 고립 처리

## 💻 Implementation Snippet (Health Score Calculator Logic)
```tsx
const HealthScoreGauge = ({ total, anomalies }: Props) => {
    const healthPercentage = ((total - anomalies) / total) * 100;
    const color = healthPercentage > 95 ? '#10b981' : healthPercentage > 80 ? '#f59e0b' : '#ef4444';

    return (
        <div className="flex flex-col items-center">
            <ResponsiveContainer width="100%" height={200}>
                <PieChart>
                    <Pie
                        data={[{ value: healthPercentage }, { value: 100 - healthPercentage }]}
                        startAngle={180} endAngle={0}
                        innerRadius={60} outerRadius={80}
                        paddingAngle={0} dataKey="value"
                    >
                        <Cell fill={color} />
                        <Cell fill="#e5e7eb" />
                    </Pie>
                </PieChart>
            </ResponsiveContainer>
            <span className="text-2xl font-bold" style={{ color }}>{healthPercentage.toFixed(1)}%</span>
            <p className="text-sm text-gray-500">Data Reliability Score</p>
        </div>
    );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 이상치 리스트가 테넌트별로 정확히 필터링되어 노출되는가?
- [ ] '승인/반려' 상태 변경 시 실시간으로 게이지 점수가 다시 계산되는가?

## 🚧 Dependencies & Blockers
- Depends on: `DQ-001`, `VIEW-018` (Data Grid)
- Blocks: 없음
