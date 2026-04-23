---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-010: 백오피스 환경설정 폼 (카페24 연동 등록)"
labels: 'frontend, view, admin, phase:5, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-010] 백오피스 탭 - 쇼핑몰 자격증명(Credentials) 등록 뷰
- 목적: B2B 화주사가 "설정 탭" 에 들어가서, 자기 회사 물류 데이터를 빨아들일 소스(카페24)의 Mall ID 와 인증 코드(`auth_code`) 를 등록하고 `ADMIN-002` API 에 쏴주는 웹 폼(Form).

## 🔗 References (Spec & Context)
- 연결 API: `ADMIN-002` (POST `/admin/integrations/shop`)
- 연관 UI/UX: 설정(Settings) 메뉴 내부 탭 구조

## ✅ Task Breakdown (실행 계획)

### 1. 연동 폼 컴포넌트 뼈대
- [ ] `src/pages/settings/IntegrationView.tsx`:
  ```tsx
  import { useForm } from 'react-hook-form';
  import { useMutation } from '@tanstack/react-query';
  import { apiClient } from '@/api/client';
  import { Button } from '@/components/ui/Button';
  import { Card } from '@/components/ui/Card';

  export const IntegrationView = () => {
    const { register, handleSubmit } = useForm();
    
    // React Query Mutation 으로 POST 요청 상태 감싸기
    const connectMutation = useMutation({
      mutationFn: (data: any) => apiClient.post('/admin/integrations/shop', data),
      onSuccess: () => alert('성공적으로 연동되었습니다! (암호화 저장됨)'),
      onError: () => alert('연동 실패. 쇼핑몰 인증 코드를 확인하세요.')
    });

    const onSubmit = (formData: any) => {
      // payload: { platform_type, platform_mall_id, auth_code }
      connectMutation.mutate({
        platform_type: "cafe24",
        platform_mall_id: formData.mallId,
        auth_code: formData.authCode
      });
    };

    return (
      <Card className="p-6 max-w-2xl">
        <h2 className="text-xl font-bold mb-4">외부 쇼핑몰 연동 관리</h2>
        <p className="text-sm text-gray-500 mb-6">
          카페24 등 사용중인 호스팅사의 API Access 정보를 등록하세요.
        </p>
        
        <form onSubmit={handleSubmit(onSubmit)} className="space-y-5">
          <div>
            <label className="block text-sm font-medium mb-1">쇼핑몰 아이디 (Mall ID)</label>
            <input 
              {...register("mallId", { required: true })}
              className="w-full bg-gray-50 border p-2 rounded-md" 
              placeholder="ex) myshop123"
            />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">OAuth 인증 코드</label>
            <input 
              {...register("authCode", { required: true })}
              className="w-full bg-gray-50 border p-2 rounded-md font-mono" 
              placeholder="발급받은 해시 코드 복사 후 붙여넣기"
            />
          </div>
          
          <Button 
             type="submit" 
             disabled={connectMutation.isPending}
          >
            {connectMutation.isPending ? '연동하는 중...' : 'Cafe24 연동하기'}
          </Button>
        </form>
      </Card>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 비동기 통신 중 버튼 블로킹 방어**
- Given: 사용자가 'Cafe24 연동하기' 버튼을 누름. 서버 통신에 3초가 걸림 (OAuth 트랜잭션).
- When: React Query 의 `isPending` 상태 변수가 True 로 토글됨.
- Then: 이 플래그에 의해 버튼이 3초간 비활성화(`disabled=true`)되고 버튼 라벨이 '연동하는 중...' 으로 바뀌어 화주사가 버튼을 다다닥 연타(Double-submit)하여 쓸데없는 서버 부하를 내는 것을 프론트엔드가 차단한다.

## ⚙️ Technical Constraints
- 실제 Cafe24 연동은 프론트엔드가 먼저 팝업창(Pop-up) 등을 띄워 로그인한 뒤 Redirect URI로부터 그 URL 파라미터(`auth_code=xxx`)를 찢어와서 보내야 하는 고도화된 흐름이다. MVP 에서는 위 스니펫처럼 수동으로 붙여넣는 창구라도 만들어 기본 구조를 타협한다.

## 🏁 Definition of Done (DoD)
- [ ] `useMutation` 훅을 이용한 로딩 상태 바인딩 제어 구조 확인?
- [ ] Card 와 폼 엘리먼트를 활용해 B2B 관리창다운 디자인 요소로 추출되었는가?
