---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Test] TEST-003: 프론트엔드 유닛 테스트 (Vitest + React Testing Library)"
labels: 'test, qa, frontend, phase:7, priority:medium'
assignees: ''
type: task
tags: [task, testing]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [TEST-003] React 컴포넌트 격리 단위 테스트(Unit Test) 셋업
- 목적: 컴포넌트 단위로 분리된 UI(예: 조건부 색상 렌더링 카드, 네비게이션 액티브 상태)가 예상한 DOM 요소를 정확하게 뱉어내는지 Node.js 환경에서 브라우저 없이 시뮬레이션 검증한다. Vite 와 궁합이 좋은 `Vitest` 를 채택한다.

## 🔗 References (Spec & Context)
- 기술 스택: `VIEW-001` (Vite 템플릿)
- 테스트 대상 타겟: `VIEW-002`, `VIEW-005` (재사용성이 높은 코어 컴포넌트 위주)

## ✅ Task Breakdown (실행 계획)

### 1. 의존성 셋업 및 환경 구성
- [ ] `npm i -D vitest @testing-library/react @testing-library/jest-dom jsdom` 명령어로 인스톨.
- [ ] `vite.config.ts` 통합:
  ```typescript
  /// <reference types="vitest" />
  import { defineConfig } from 'vite';
  import react from '@vitejs/plugin-react';

  export default defineConfig({
    plugins: [react()],
    test: {
      environment: 'jsdom', // 브라우저 없는 노드 환경에서 DOM 흉내
      setupFiles: ['./src/setupTests.ts'],
      globals: true
    }
  });
  ```

### 2. Jest-dom Matcher 주입
- [ ] `src/setupTests.ts`:
  ```typescript
  import '@testing-library/jest-dom';
  // 이를 통해 expect(element).toHaveClass('text-danger') 등 직관적 단언(assert) 가능
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: `cn()` 유틸 함수 고립 텍스트 검사 (Logic Test)**
- Given: Tailwind 클래스를 묶어주는 `src/utils/cn.ts` (twMerge 래퍼) 가 주어진다.
- When: `expect(cn("bg-red-500 text-white", "bg-blue-500")).toBe("text-white bg-blue-500")` 를 돌린다.
- Then: `vitest` 스크립트를 통해 성공(Passed) 응답을 반환하여, 추후 디자인 공통 로직이 파괴되지 않음을 보수성 있게 증명해낸다.

## ⚙️ Technical Constraints
- 프론트엔드 UI 테스트는 변경이 잦으므로 너무 포괄적인 화면 단위 전체 렌더링보다는, 룰이 변하지 않는 "순수 함수(Utils)"나 "디자인 시스템 컴포넌트(Atomic)" 위주로 작성하여 깨짐을 최소화 한다.

## 🏁 Definition of Done (DoD)
- [ ] `package.json` 에 `npm run test` vitest 트리거 구문 삽입?
- [ ] DOM 환경을 지원하는 `jsdom` 파라미터가 vite.config에 포함되었는가?
