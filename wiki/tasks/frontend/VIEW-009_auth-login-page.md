---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-009: 인증 (Auth) 로그인 폼 컴포넌트 구현"
labels: 'frontend, view, auth, phase:5, priority:critical'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-009] B2B 관리자 로그인 뷰 및 JWT 발급/저장 로직
- 목적: SaaS 시스템에 접근하기 위한 관문 프론트엔드 제작. `react-hook-form`을 이용하여 유효성 검사를 진행하고 백엔드(`AUTH-001`)로 아이디/비밀번호를 보내 `Access`토큰을 LocalStorage 등에 저장 후 Dashboard 로 리다이렉트 시킨다.

## 🔗 References (Spec & Context)
- 인증 API: `AUTH-001` (POST `/api/v1/auth/login`)
- 전역 상태/라우터: React Router-dom 의 `Navigate` 기능 연계

## ✅ Task Breakdown (실행 계획)

### 1. Form 유효성 검사 셋업
- [ ] `npm i react-hook-form @hookform/resolvers yup` 등 폼 유효성 패키지 셋팅.
- [ ] `src/pages/LoginPage.tsx`:
  ```tsx
  import { useForm } from 'react-hook-form';
  import { useNavigate } from 'react-router-dom';
  import { apiClient } from '@/api/client';
  import { Button } from '@/components/ui/Button';

  export const LoginPage = () => {
    const navigate = useNavigate();
    const { register, handleSubmit, formState: { errors } } = useForm();

    const onSubmit = async (data: any) => {
      try {
        // FastAPI OAuth2 표준은 application/x-www-form-urlencoded 를 많이 사용함
        const formData = new URLSearchParams();
        formData.append('username', data.email);
        formData.append('password', data.password);
        
        const res = await apiClient.post('/auth/login', formData, {
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
        });
        
        // Storage 보관 (VIEW-003 인터셉터가 파싱하게 됨)
        localStorage.setItem('access_token', res.data.access_token);
        localStorage.setItem('refresh_token', res.data.refresh_token);
        
        navigate('/dashboard'); // 진입
      } catch (err) {
        alert('이메일 혹은 비밀번호가 틀렸습니다.');
      }
    };

    return (
      <div className="flex h-screen items-center justify-center bg-gray-50">
        <div className="w-full max-w-md p-8 bg-white rounded-xl shadow-lg">
          <h1 className="text-2xl font-bold text-center mb-6">물류 AI 수요예측 포탈</h1>
          <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
            <div>
              <label className="block text-sm font-medium mb-1">업무용 이메일</label>
              <input 
                 {...register("email", { required: true })} 
                 className="w-full p-2 border rounded-md" 
                 type="email" 
              />
            </div>
            <div>
              <label className="block text-sm font-medium mb-1">비밀번호</label>
              <input 
                 {...register("password", { required: true })} 
                 className="w-full p-2 border rounded-md" 
                 type="password" 
              />
            </div>
            <Button type="submit" className="w-full mt-4" size="lg">로그인</Button>
          </form>
        </div>
      </div>
    );
  };
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 성공적인 토큰 저장 및 라우팅**
- Given: 올바른 ID와 비밀번호를 입력함
- When: `onSubmit` 버튼을 눌러 비동기 200 OK 응답을 받음
- Then: 브라우저 DevTools 의 `Application -> Local Storage` 영역에 `access_token` 키가 무사히 박히고 즉각 `/dashboard` URL로 경로가 바뀌어 로그인 장벽(Guard)을 통과해 메인 화면을 보게 된다.

## ⚙️ Technical Constraints
- 폼 자체의 `onSubmit(e)` 기본 동작 방지(`e.preventDefault()`)를 `react-hook-form` 이 대신 해주므로, 네이티브한 Action 전송이 일어나지 않도록 제어 코드를 위 스니펫대로 따른다.
- 백엔드가 OAuth2 `OAuth2PasswordRequestForm` 의존성을 사용할 경우 Body 는 json이 아닌 `url-encoded` 폼이어야 함을 필히 숙지한다.

## 🏁 Definition of Done (DoD)
- [ ] `react-hook-form` 을 활용한 이메일/비밀번호 바인딩 적용?
- [ ] 로그인 성공 후 Local Storage JWT 세팅 및 `navitage()` 로 라우트 이동 구현?
