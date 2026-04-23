---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-015: 에러 바운더리(Error Boundary) 방어막 및 통합 스켈레톤(Skeleton)"
labels: 'frontend, view, ux, phase:9, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-015] 단일 컴포넌트 크래시에 대비한 React Error Boundary 파티션
- 목적: 카드 위젯 하나가 백엔드 파라미터 에러로 죽었을 때, `React` 파서 전체가 무너지며 렌더 트리가 날아가는 `White Screen of Death (흰 화면)` 현상을 차단한다. 오류난 카드 1곳만 "데이터 장애" 마크를 띄우고 전체 대시보드는 멀쩡하게 작동시키는 우수한 엔터프라이즈 UX를 구성한다.

## 🔗 References (Spec & Context)
- 적용 구역: `VIEW-004` (Dashboard Layout의 Outlet 최상단 등 부분 적용)
- 연동 툴: `react-error-boundary` 라이브러리 권장

## ✅ Task Breakdown (실행 계획)

### 1. 전역 Fallback 컴포넌트 확보
- [ ] `src/components/ui/ErrorFallbackCard.tsx`:
  ```tsx
  import { FallbackProps } from 'react-error-boundary';
  import { Button } from './Button';

  export const ErrorFallbackCard = ({ error, resetErrorBoundary }: FallbackProps) => {
    return (
      <div className="p-6 bg-red-50 border border-red-200 rounded-lg flex flex-col items-center">
        <span className="text-2xl mb-2">⚠️</span>
        <h3 className="font-semibold text-red-700">해당 패널을 불러올 수 없습니다</h3>
        <p className="text-sm text-red-500 mb-4">{error.message}</p>
        <Button variant="outline" size="sm" onClick={resetErrorBoundary}>
          다시 불러오기
        </Button>
      </div>
    );
  };
  ```

### 2. Layout과 Query 혼합 블록킹 적용
- [ ] `src/pages/DashboardPage.tsx` 내부:
  ```tsx
  import { ErrorBoundary } from 'react-error-boundary';
  import { Suspense } from 'react';
  
  // Skeleton 컴포넌트는 회색박스가 깜빡이는 디자인(UX)
  import { DashboardSkeleton } from '@/components/ui/Skeleton';
  import { XAIPanel } from '@/components/dashboard/XAIPanel'; // VIEW-007

  export const DashboardScreen = () => {
    return (
      <div className="space-y-6">
        <ErrorBoundary FallbackComponent={ErrorFallbackCard}>
          <Suspense fallback={<DashboardSkeleton height="300px" />}>
             {/* 비동기 렌더링에 취약한 분석 차트 블록들 */}
             <XAIPanel sku="ALL" targetDate="2026-11-11" />
          </Suspense>
        </ErrorBoundary>
        
        <ErrorBoundary FallbackComponent={ErrorFallbackCard}>
          <WorkforceTable />
        </ErrorBoundary>
      </div>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 부분 컴포넌트 장애 고립 (Isolation)**
- Given: XAI 분석 차트를 가져오는 `/metadata/explain` API가 백엔드 오류 500을 뿜어 React가 `Undefined` 렌더링을 쳤음.
- When: React Error Boundary가 이를 가로챔
- Then: 전체 화면이 날아가는 대신, XAI 차트 자리 박스 영역만 깔끔하게 `ErrorFallbackCard` (빨간 경고박스) 로 스위칭되며, 바로 아래에 있는 **권장 인력 계산 표(Workforce Table)는 전혀 영항을 받지 않고 온전하게 기능 연산을 성공**해 낸다.

## ⚙️ Technical Constraints
- `react-query` 와 `ErrorBoundary` 를 연동하려면, 쿼리 훅 옵션에 `useErrorBoundary: true` (버전에 따라 `throwOnError: true`) 를 무조건 켜주어야 React가 생명주기 레벨에서 에러를 낚아채 Fallback을 발동시킨다.

## 🏁 Definition of Done (DoD)
- [ ] 컴포넌트별 렌더 트리를 감싸는 `<ErrorBoundary>` 바인딩 셋업 완료?
- [ ] API 지연을 부드럽게 넘기는 `<Suspense fallback={...}>` (스켈레톤) 조립 유무?
