---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-001: 프론트엔드 프로젝트 뼈대 및 SPA 아키텍처 설정"
labels: 'frontend, view, setup, phase:5, priority:critical'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-001] React 기반 프론트엔드 프로젝트 초기화 및 환경 설정
- 목적: B2B 웹 애플리케이션 화면 구성을 위해, 성능이 뛰어난 React SPA (Vite 기반) 프로젝트를 세팅하고, Tailwind CSS 기반의 스타일링 환경 및 계층형 폴더 구조를 확립한다.

## 🔗 References (Spec & Context)
- SRS 뷰/컴포넌트: [`SRS-V1.0.md#§5.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — UI 화면 프레임워크 명세
- 기술 스택 제약: C-TEC-001 (Streamlit 은 테스트용이며, 본 문서는 상용 MVP 배포용 프론트엔드로 React 스택을 고려할 경우의 표준을 지정함. Streamlit을 쓸 경우에도 이에 준하는 폴더 분리 적용).

## ✅ Task Breakdown (실행 계획)

### 1. Vite + React + TypeScript 초기화
- [ ] 프론트엔드 루트 디렉토리(`./frontend`)에서 `npm create vite@latest . -- --template react-ts` 실행.
- [ ] 필수 의존성 설치: 
  `npm install axios @tanstack/react-query react-router-dom lucide-react recharts clsx tailwind-merge`

### 2. Tailwind CSS 인스톨 및 Config
- [ ] `tailwind.config.js` 에 B2B 테마 토큰 주입:
  ```javascript
  /** @type {import('tailwindcss').Config} */
  export default {
    content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
    theme: {
      extend: {
        colors: {
          primary: "#1E3A8A", // 기업용 딥블루 (Brand)
          secondary: "#64748B",
          success: "#10B981", // 증가/성공
          danger: "#EF4444", // 감소/경고
          surface: "#FFFFFF",
          background: "#F8FAFC", // 아주 옅은 회색 배경
        },
        fontFamily: {
          sans: ['Inter', 'Pretendard', 'sans-serif'], // 모던 B2B 폰트
        }
      },
    },
    plugins: [],
  }
  ```

### 3. 디렉토리 구조 확립 (Feature-Sliced Design 응용)
- [ ] `src/` 하위에 책임을 분리한 폴더 구조 뼈대 커밋:
  - `/api` : Axios 인스턴스 및 백엔드 Fetch 함수
  - `/components` : UI 단위 조각 (Button, Card, Modal)
  - `/hooks` : React-Query 를 래핑한 커스텀 훅 (ex: `useForecast.ts`)
  - `/pages` : 라우팅되는 실제 페이지 화면 (Login, Dashboard, Settings)
  - `/store` : Zustand 등 전역 상태 관리 (필요 시)
  - `/utils` : 날짜/숫자 포맷 유틸 (ex: 천단위 콤마 `formatNumber`)

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Tailwind CSS 작동 및 폰트 컴파일 확인**
- Given: `App.tsx` 최상단에 `<div className="bg-background text-primary font-sans text-2xl">Tailwind Test</div>` 렌더링
- When: `npm run dev` 를 쳐서 로컬 호스트에 접속함
- Then: 브라우저에 파란색(Primary) 텍스트와 회색 배경이 먹힌 화면이 출력되며 클래스 네임 스캐닝 버그가 없음을 증명한다.

## ⚙️ Technical Constraints
- `clsx` 와 `tailwind-merge` 를 묶은 `cn()` 유틸리티를 `src/utils/cn.ts` 에 작성하여, 이후 컴포넌트들에서 클래스네임 충돌 없이 동적 스타일링을 할 수 있게 마련할 것.

## 🏁 Definition of Done (DoD)
- [ ] Vite+React+TS 인스톨 완료?
- [ ] Tailwind Config 및 폰트/컬러 테마 토큰 세팅 완료?
- [ ] 확장 가능한 `src/` 디렉토리 구조 커밋?
