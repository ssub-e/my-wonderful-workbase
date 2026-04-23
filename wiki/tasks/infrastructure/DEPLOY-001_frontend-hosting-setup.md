---
name: Deployment Task
about: SRS 기반 MVP 실운영 프로덕트 릴리즈 설정
title: "[Deploy] DEPLOY-001: 프론트엔드 호스팅 및 CD(지속 배포) 환경 셋업"
labels: 'infra, deploy, frontend, phase:6, priority:critical'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DEPLOY-001] React/Vite SPA 프론트엔드 환경 호스팅 배포(Production)
- 목적: 로컬호스트(localhost:5173)를 벗어나 실제 B2B 고객들이 `https://dashboard.b2bai.com` 과 같은 커스텀 도메인으로 접속할 수 있는 프론트엔드 상용 서비스망을 구축한다. 

## 🔗 References (Spec & Context)
- 기술 스택: `VIEW-001` (Vite, React 기반)
- CI/CD 전략: Vercel 또는 AWS S3 + CloudFront 등 CDN 배포 플랫폼 (비용 / 속도 측면 Vercel 권장)

## ✅ Task Breakdown (실행 계획)

### 1. Build 스크립트 점검
- [ ] `frontend/package.json`: 
  ```json
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  }
  ```
  - TypeScript 에러가 있으면 빌드가 강제로 중단되도록 `tsc` 선행 커맨드가 포함되어 있는지 확인한다.

### 2. Vercel 플랫폼 연동 (혹은 동급 CDN)
- [ ] GitHub 저장소의 `main` 또는 `prod` 브랜치를 Vercel 프로젝트에 연동한다.
- [ ] Build Command: `npm run build`
- [ ] Output Directory: `dist`
- [ ] 핵심 셋업: Vercel(또는 Nginx) 환경 변수를 통해, API 서버 요청 주소를 프록싱한다. (SPA 특성상 Vercel에서 새로고침 시 404가 나는 걸 막고자 Rewrite 룰 `vercel.json` 세팅 필수).

### 3. Vercel용 라우트 재작성 지시서 
- [ ] `frontend/vercel.json` 추가:
  ```json
  {
    "rewrites": [
      { "source": "/api/v1/(.*)", "destination": "https://[여러분의-백엔드-API-도메인]/api/v1/$1" },
      { "source": "/(.*)", "destination": "/index.html" }
    ]
  }
  ```
  - 첫 줄: 프론트엔드가 자기를 향해 날린 API 콜을 전부 백엔드 도메인(EC2 등)으로 토스(CORS 문제 회피의 정석).
  - 둘째 줄: 라우팅이 잡히지 않는 모든 SPA 내부 경로를 `index.html` 로 돌려 React Router 가 파싱하도게 위임.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: SPA 클라이언트 측 라우팅 (새로고침 방어)**
- Given: 사용자가 배포된 `https://domain.com/settings/integration` 이라는 중간 Path 주소를 브라우저 주소창에 복사해 직접 치고 돌어옴.
- When: 웹 서버(Vercel or AWS)가 요청을 받음
- Then: 서버는 `/settings` 라는 폴더 구조가 실제로 없어도 404 에러를 내뱉지 않고 `index.html` 을 내어주어 React Router 가 렌더링되게 하는 SPA Catch-all 라우팅이 정상 동작한다.

## ⚙️ Technical Constraints
- API 목적지(Destination) 프로토콜은 반드시 `HTTPS` 가 적용된 서버여야 만 브라우저의 Mixed Content 차단에 걸리지 않는다 (프론트는 HTTPS인데 백엔드가 HTTP일 경우 네트워크 락 다운 발생).

## 🏁 Definition of Done (DoD)
- [ ] `tsc` 기반 빌드 에러 체커가 Build 스크립트에 포함되어 있는가?
- [ ] SPA(Single Page Application) 전용 라우팅 룰(`vercel.json` 등) 환경 제어 파일 커밋?
- [ ] 외부 API 서버로 트래픽을 프록싱하는 룰이 셋업되었나?
