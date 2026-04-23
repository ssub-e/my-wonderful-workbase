---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-012: 관리자 권한 모달 폼 (직원 초대 / RBAC)"
labels: 'frontend, view, admin, auth, phase:5, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-012] 직원 초대 및 권한 직무(Role) 할당 모달/폼
- 목적: B2B 시스템 답게 "우리 회사 다른 담당자에게 아이디를 열어주기" 위한 폼 UI. 이메일과 직무 등급(Role-Admin, Role-MD 등)을 라디오 버튼 등으로 골라 `ADMIN-004` 로 POST를 날려서 신규 유저 계정을 생성시켜 준다.

## 🔗 References (Spec & Context)
- 기반 API: `ADMIN-004` (POST `/admin/users/invite`)
- 보안 구조: 본 API가 `["ADMIN"]` 에게만 열려있으므로, 프론트에서도 `Current_User.Role === "ADMIN"` 이 아닐시 이 컴포넌트 자체 노출을 Hide 시키는 분기 렌더링이 필요.

## ✅ Task Breakdown (실행 계획)

### 1. 직원 초대 폼 모달
- [ ] `src/components/settings/UserInviteForm.tsx`:
  ```tsx
  import { useForm } from 'react-hook-form';
  import { useMutation } from '@tanstack/react-query';
  import { apiClient } from '@/api/client';
  import { Button } from '@/components/ui/Button';

  export const UserInviteForm = () => {
    const { register, handleSubmit, reset } = useForm();
    
    const inviteUser = useMutation({
      mutationFn: (payload: any) => apiClient.post('/admin/users/invite', payload),
      onSuccess: () => {
        alert('요청하신 직원의 계정이 생성되었습니다.');
        reset(); // 폼 초기화 
      }
    });

    const onSubmit = (d: any) => inviteUser.mutate({
      name: d.name,
      email: d.email,
      role: d.role,
      temporary_password: d.password
    });

    return (
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-4 p-4 border rounded-md">
        <h3 className="font-semibold text-lg">신규 서브 계정 추가 (관리자용)</h3>
        
        <div className="grid grid-cols-2 gap-4">
          <input {...register("name")} placeholder="이름 (ex. 김대리)" className="border p-2 rounded" />
          <input {...register("email")} placeholder="업무용 이메일" type="email" className="border p-2 rounded" />
          <input {...register("password")} placeholder="부여할 임시 비밀번호" type="password" className="border p-2 rounded" />
          
          <select {...register("role")} className="border p-2 rounded bg-white">
            <option value="ADMIN">ADMIN (권한 전체 통제)</option>
            <option value="CENTER_MANAGER">CENTER_MANAGER (물류센터장)</option>
            <option value="MD">MD (결제/상품 기획자)</option>
          </select>
        </div>
        
        <Button disabled={inviteUser.isPending} className="w-full">
          사용자 계정 생성
        </Button>
      </form>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 성공 시 폼 잔여물 비우기 (UX)**
- Given: 관리자가 "김철수, chulsu@a.com, 비번" 쳐 놓고 계정 생성을 눌렀음
- When: 비동기 응답(`onSuccess`) 이 200 을 반환하며 완료됨
- Then: `reset()` 함수가 동작하여 즉각 모달 내부의 텍스트 필드 4개가 싹 다 하얀색 공란으로 되돌아가며, 실수로 두 번 눌려 Duplicate 에러가 나는 불상사를 막는 우수한 UX/UI 상호작용 피드백을 제공한다.

## ⚙️ Technical Constraints
- HTML Native `<select>` 문법 대신 디자인 고도화를 꾀하기 위해, 추후 Shadcn의 `<Select>` 나 Headless UI `<Listbox>` 컴포넌트로 포팅할 것을 대비하여 상태는 `react-hook-form` এর `Controller` 패턴으로 감싸 확장성을 보장한다.

## 🏁 Definition of Done (DoD)
- [ ] 이름/이메일/비밀번호/Role 의 4가지 파라미터를 받는 폼 작성?
- [ ] 권한 분기 선택 박스(`select`) 구성 유무?
- [ ] 폼 Submit 직후 `reset()` 초기화 함수 보장 적용?
