---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-003: API 클라이언트 (Axios) 및 React Query 연동"
labels: 'frontend, view, api, phase:5, priority:critical'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-003] 인증 토큰 자동 갱신(Interceptor) 및 전역 서버 상태 관리(React Query) 셋업
- 목적: 컴포넌트들이 직접 `fetch`를 날리면서 Loading과 Error 처리에 렌더링을 뺏기는 것을 막기 위해, 중앙 `Axios` 클라이언트를 두고 `@tanstack/react-query` 를 이용하여 자동 캐싱-재시도-로딩 처리를 관리한다.

## 🔗 References (Spec & Context)
- 인증 흐름 연동: `AUTH-001` (JWT Access / Refresh 흐름)
- 타겟 API: 백엔드 `DASH`, `ADMIN` 네임스페이스 전체

## ✅ Task Breakdown (실행 계획)

### 1. Axios 인스턴스 및 인터셉터(Interceptor)
- [ ] `src/api/client.ts`:
  ```typescript
  import axios from 'axios';

  export const apiClient = axios.create({
    baseURL: import.meta.env.VITE_API_BASE_URL || '/api/v1',
    timeout: 10000,
  });

  // Request: 매 요청마다 LocalStorage의 JWT를 헤더에 박음
  apiClient.interceptors.request.use((config) => {
    const token = localStorage.getItem('access_token');
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
  });

  // Response: 401 만료 시 자동 Refresh 로직
  apiClient.interceptors.response.use(
    (response) => response,
    async (error) => {
      const originalRequest = error.config;
      if (error.response?.status === 401 && !originalRequest._retry) {
        originalRequest._retry = true;
        try {
          // Refresh API 호출 (AUTH-001 로직)
          const refresh_token = localStorage.getItem('refresh_token');
          const resp = await axios.post('/api/v1/auth/refresh', { refresh_token });
          localStorage.setItem('access_token', resp.data.access_token);
          // 토큰 교체 후 원본 API 다시 날림
          return apiClient(originalRequest);
        } catch (e) {
          // 리프레시마저 실패하면 완전 퇴출
          localStorage.clear();
          window.location.href = '/login';
        }
      }
      return Promise.reject(error);
    }
  );
  ```

### 2. React Query Provider 바인딩
- [ ] `src/main.tsx` 최상단:
  ```tsx
  import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        refetchOnWindowFocus: false, // 탭 전환 시점 불필요한 API 폭증 방오
        retry: 1, // 실패시 1회 자동 재시도
      },
    },
  });

  // <QueryClientProvider client={queryClient}> <App /> </QueryClientProvider>
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Access Token 만료에 따른 자동 재발급 투명성**
- Given: 유저의 AccessToken 이 30분 기한을 넘겨 만료됨 (단, Refresh는 유효함)
- When: 대시보드 화면이 로드되면서 `apiClient.get('/dashboard/summary')` 를 호출함
- Then: 백엔드가 401 을 뱉으나 Axios Interceptor가 이를 낙아채 백그라운드에서 `/refresh` 요청으로 새 토큰을 꺼내온 뒤 원래 요청을 200 OK로 이어서 반환한다. (사용자는 화면이 끊기는 것을 전혀 느끼지 못함).

## ⚙️ Technical Constraints
- Refresh 토큰마저 만료된 완전 퇴출 시점(redirect)에선 남아있는 Auth State(전역 스토어)도 깔끔히 비워주어야 불완전한 렌더링이 발생하지 않는다.

## 🏁 Definition of Done (DoD)
- [ ] Header에 JWT 토큰을 찔러넣는 Axios Request Interceptor 작성?
- [ ] 401 에러를 Catch해 자동 토큰을 교체하는 Response Interceptor 작성?
- [ ] 프로젝트 Root에 React Query Provider 랩핑?
