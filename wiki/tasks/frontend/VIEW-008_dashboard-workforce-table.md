---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-008: 일일 물류 인력 권장 배치표 (Data Grid) 컴포넌트"
labels: 'frontend, view, dashboard, phase:5, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-008] 대시보드 하단 - 센터별 인력 추천 및 알림톡 상태 Data Grid 영역
- 목적: `DASH-004` 배포 API를 활용하여, 내일 호출해야 할 아르바이트생 수와 현재 알림톡 발송 완료 여부를 파악하기 쉬운 표(Table) 형태로 렌더링한다.

## 🔗 References (Spec & Context)
- 연결 API: `DASH-004` (GET `/dashboard/workforce`)
- UI 뷰: [대시보드 메인 페이지 하단 영역]

## ✅ Task Breakdown (실행 계획)

### 1. Data Fetching Hook
- [ ] `src/hooks/useWorkforcePanel.ts` 작성 (React-Query를 활용, target_date를 의존성 배열로 삼아 날짜 변경 시 자동 리패치).

### 2. Table 컴포넌트 구현
- [ ] `src/components/dashboard/WorkforceTable.tsx`:
  ```tsx
  import { useWorkforcePanel } from '@/hooks/useWorkforcePanel';
  import { Card } from '@/components/ui/Card';
  import { Badge } from '@/components/ui/Badge'; // VIEW-002 에서 구현 예정

  export const WorkforceTable = ({ targetDate }: { targetDate: string }) => {
    const { data, isLoading } = useWorkforcePanel(targetDate);

    if (isLoading) return <div>표를 불러오는 중...</div>;

    return (
      <Card className="overflow-hidden">
        <div className="p-4 border-b bg-gray-50 flex justify-between items-center">
          <h3 className="font-semibold text-gray-700">창고/센터별 인력 배치 권장표 ({targetDate})</h3>
        </div>
        <div className="overflow-x-auto">
          <table className="w-full text-left text-sm whitespace-nowrap">
            <thead className="bg-gray-100/50 text-gray-500">
              <tr>
                <th className="px-6 py-3 font-medium">물류센터명</th>
                <th className="px-6 py-3 font-medium">내일 예상 출고(Box)</th>
                <th className="px-6 py-3 font-medium text-primary">필요 권장 인력</th>
                <th className="px-6 py-3 font-medium">센터장 발송 상태</th>
              </tr>
            </thead>
            <tbody className="divide-y divide-gray-200">
              {data?.data.map((row: any) => (
                <tr key={row.center_id} className="hover:bg-gray-50 transition-colors">
                  <td className="px-6 py-4 font-medium text-gray-900">{row.center_id}</td>
                  <td className="px-6 py-4">{row.predicted_total_shipments.toLocaleString()} 건</td>
                  <td className="px-6 py-4 font-bold text-primary">{row.recommended_workers} 명</td>
                  <td className="px-6 py-4">
                    {row.notification_sent ? (
                      <Badge variant="success">발송 완료</Badge>
                    ) : (
                      <Badge variant="outline">대기중</Badge>
                    )}
                  </td>
                </tr>
              ))}
              {data?.data.length === 0 && (
                <tr><td colSpan={4} className="text-center py-6 text-gray-400">데이터가 없습니다.</td></tr>
              )}
            </tbody>
          </table>
        </div>
      </Card>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 상태값 Boolean 뱃지 치환**
- Given: 백엔드에서 넘어온 `notification_sent` 가 True 로 세팅된 행과 False인 행 혼재.
- When: Table의 렌더 함수가 루프를 돎
- Then: True인 열은 내부적으로 초록색 배경의 "발송 완료" `Badge` 로 그려지고, False 인 열은 회색/투명 테두리의 "대기중" 으로 그려져 표의 직관성을 보장한다.

## ⚙️ Technical Constraints
- 화면의 가로 폭이 줄어들면 Table이 깨지는 현상을 방어하기 위해 부모를 `overflow-x-auto` 로 감싸 브라우저 외부 대신 컴포넌트 내부에서만 가로 스크롤이 돌게 한다.

## 🏁 Definition of Done (DoD)
- [ ] 기본 표(Table) 태그 뼈대와 Tailwind 스타일이 조합되었는가?
- [ ] 배열 렌더링(`map`)과 `colSpan` Empty State 보강 처리가 되어 있는가?
