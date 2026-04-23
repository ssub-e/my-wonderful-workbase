---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-011: 백오피스 환경설정 폼 (물류센터 CAPA 조절)"
labels: 'frontend, view, admin, phase:5, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-011] 백오피스 탭 - 창고별 작업량(CAPA) 조절 폼
- 목적: B2B 시스템 관리자가 언제든 자신의 물류센터 숙련도를 가감할 수 있는 Slider 혹은 Number Input 뷰. 이 값을 수정하면 `ADMIN-003` API를 건드려 다음날부터 예측 인원이 똑똑해지게 통제한다.

## 🔗 References (Spec & Context)
- 연결 API: `ADMIN-003` (PATCH `/admin/centers/capacity`)
- 종속 기능: `VIEW-008` (표에 나타날 추천명수에 영향을 주는 가장 딥한 변수 설정폼)

## ✅ Task Breakdown (실행 계획)

### 1. CAPA 설정 뷰
- [ ] `src/pages/settings/CapacityView.tsx`:
  ```tsx
  import { useForm } from 'react-hook-form';
  import { useMutation } from '@tanstack/react-query';
  import { apiClient, queryClient } from '@/api/client';
  import { Button } from '@/components/ui/Button';
  import { Card } from '@/components/ui/Card';

  export const CapacityView = () => {
    const { register, handleSubmit } = useForm({
      defaultValues: { capacity: 50 }, // 초기값 50 박스
    });
    
    // PATCH 전송
    const updateCapa = useMutation({
      mutationFn: (capa: number) => 
        apiClient.patch('/admin/centers/capacity', { center_name: "기본센터", capacity_per_worker: capa }),
      onSuccess: () => {
        alert('반영되었습니다. 익일 역산부터 적용됩니다.');
        // Workforce 관련 캐시 날려서 화면 즉시 갱신 대비
        queryClient.invalidateQueries({ queryKey: ['workforce'] }); 
      }
    });

    return (
      <Card className="p-6 max-w-2xl mt-8">
        <h2 className="text-xl font-bold mb-4">현장 인력(CAPA) 기준 조율</h2>
        <p className="text-sm text-gray-500 mb-6">
          이 예측기가 '작업자 1명'을 몇 개의 출고 박스로 취급할지 기준값을 입력합니다. 
          컨베이어 벨트 도입 등으로 생산성이 오르면 상향 조절하시기 바랍니다.
        </p>
        
        <form onSubmit={handleSubmit((d) => updateCapa.mutate(Number(d.capacity)))} className="flex items-end gap-4">
          <div className="flex-1">
            <label className="block text-sm font-medium mb-1">1인당 하루 처리 가능 박스 수량</label>
            <input 
              {...register("capacity", { min: 1 })}
              type="number"
              className="w-full bg-gray-50 border p-2 rounded-md" 
            />
          </div>
          <Button type="submit" variant="outline" disabled={updateCapa.isPending}>
            반영 및 저장
          </Button>
        </form>
      </Card>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 성공 시 화면 전역 캐시 무효화 (Invalidate Queries)**
- Given: 관리자가 CAPA를 50에서 100으로 바꾼 직후, 대시보드의 '필요 예상인력' 화면으로 되돌아감
- When: PATCH API 성공 시 호출된 `queryClient.invalidateQueries` 가 동작함
- Then: 이 캐시 파괴 트랜잭션 덕분에 다른 화면으로 넘어가자마자 `useWorkforcePanel` 훅이 데이터를 스스로 다시 긁어와 (Refetch), 바뀐 세팅값을 바탕으로 재역산된 인원수 표를 즉각 볼 수 있어 SPA 환경(UX)의 매끄러움을 극대화한다.

## ⚙️ Technical Constraints
- HTML 네이티브 `type="number"`를 쓰더라도 폼 데이터는 String으로 파싱될 수 있으므로 전송 전에 `Number()` 형변환 래퍼가 올바르게 작동하는지 `react-hook-form` 디버그를 돌려본다.

## 🏁 Definition of Done (DoD)
- [ ] Number 인풋 폼과 PATCH Mutation이 작성 확인 완료?
- [ ] `queryClient.invalidateQueries` 를 활용한 프론트엔드 캐시 폐기 로직 점검?
