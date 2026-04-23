---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[SaaS-Admin] SUPER-003: SaaS 파트너 리스트 관리 (Super Admin Grid) 뷰"
labels: 'frontend, view, saas-admin, phase:8, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SUPER-003] SaaS 내부 백오피스 전용 (루트 계정용) 고객사 관리 대시보드
- 목적: B2B 시스템을 서비스하는 주체(본사 영업/개발팀)가 현재 가입된 화주사들의 리스트, 남은 요금제 통계, 사용 정지 버튼을 클릭만으로 통제할 수 있는 숨겨진 어드민 뷰 화면이다.

## 🔗 References (Spec & Context)
- 연결 라우터: `SUPER-002` (SaaS Control API)
- 프레임워크: `VIEW-013` (페이지네이션 Data Grid 뼈대 재활용)

## ✅ Task Breakdown (실행 계획)

### 1. View 라우팅 및 렌더 가드
- [ ] `src/pages/saas/PartnerListScreen.tsx`:
  - 루트 경로 `/saas/partners` 등에 배치.
  - Context/Store에 보관된 `currentUser.role` 이 `SUPER_ADMIN` 이 아닐 시 냅다 `/` 로 튕겨내거나 404 컴포넌트를 띄워 일반 화주사들의 호기심 접근을 물리적으로 잘라낸다.

### 2. 제어용 Data Grid 컴포넌트 
- [ ] `src/components/saas/TenantManagerTable.tsx`:
  ```tsx
  import { useQuery, useMutation } from '@tanstack/react-query';
  import { apiClient, queryClient } from '@/api/client';
  import { Button } from '@/components/ui/Button';

  export const TenantManagerTable = () => {
    // 테넌트 리스트 전체 조회 (Super Admin 전용 함수)
    const { data } = useQuery({ queryKey: ['saas-tenants'], queryFn: () => apiClient.get('/saas/tenants') });
    
    // 상태 변경 결재 (SUPER-002 연결)
    const suspendMutation = useMutation({
      mutationFn: ({ id, status }) => apiClient.patch(`/saas/tenants/${id}/status`, { new_status: status }),
      onSuccess: () => queryClient.invalidateQueries({ queryKey: ['saas-tenants'] })
    });

    return (
      <table className="min-w-full divide-y border">
        <thead>
          <tr><th>화주사명</th><th>요금제(Plan)</th><th>현재 상태</th><th>운영 통제(Action)</th></tr>
        </thead>
        <tbody>
          {data?.map(tenant => (
            <tr key={tenant.id}>
              <td>{tenant.company_name}</td>
              <td className="font-mono">{tenant.plan}</td>
              <td>{tenant.status === 'ACTIVE' ? '🟢 정상' : '🔴 정지'}</td>
              <td>
                 {tenant.status === 'ACTIVE' ? (
                   <Button variant="danger" size="sm" onClick={() => suspendMutation.mutate({id: tenant.id, status: 'SUSPENDED'})}>
                     계정 중지
                   </Button>
                 ) : (
                   <Button variant="outline" size="sm" onClick={() => suspendMutation.mutate({id: tenant.id, status: 'ACTIVE'})}>
                     정지 해제
                   </Button>
                 )}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 동적 액션 버튼(Toggle) 피드백 렌더링**
- Given: 상태가 '정지' 인 불량 고객사 Row 가 존재함.
- When: 운영자가 <정지 해제> 버튼을 누름.
- Then: `suspendMutation` 이 무사히 서버 통신을 끝내면 `invalidateQueries` 가 동작하여 표가 스스로 리프레시되고, 이 회사가 다시 '🟢 정상' 마크 및 <계정 중지> 버튼 액션으로 변경되어 매끄러운 영업/CS 운영을 돕는다.

## ⚙️ Technical Constraints
- 이 뷰는 반드시 B2B 본사 담당자 1~2명만 보는 화면이므로 미려한 디자인보다 **검색, 오름/내림차순 정렬, 빠른 속도** 등 관리 툴로서의 펑셔널한 요건이 우선된다.

## 🏁 Definition of Done (DoD)
- [ ] 권한 가드(`if role !== 'SUPER_ADMIN' return Null`) 통제 장치 존재 여부?
- [ ] Tenant 정지 및 해제를 위한 Toggle 형태의 Request 로직 작성완료?
