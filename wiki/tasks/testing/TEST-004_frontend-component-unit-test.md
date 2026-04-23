---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Test] TEST-004: 데이터 표출형 컴포넌트 조건부 렌더링 QA 테스팅"
labels: 'test, qa, frontend, phase:7, priority:medium'
assignees: ''
type: task
tags: [task, testing]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [TEST-004] 대시보드 KPI 카드 및 증감률 조건부 색상 렌더링 단위 테스트
- 목적: `VIEW-005` 에서 구현한 "마이너스 값이면 빨간색, 플러스면 파란색/초록색" 이 되는 B2B 핵심 지표(Stat) 위젯의 조건부 렌더링 로직을 `@testing-library/react` 로 분리하여 굳힌다.

## 🔗 References (Spec & Context)
- 타겟 컴포넌트: `VIEW-005` 내 `StatCard.tsx`
- 활용 프레임워크: `TEST-003` (Vitest)

## ✅ Task Breakdown (실행 계획)

### 1. StatCard 감소(Danger) 케이스 렌더링 테스트
- [ ] `src/components/dashboard/__tests__/StatCard.test.tsx`:
  ```tsx
  import { render, screen } from '@testing-library/react';
  import { StatCard } from '../StatCard';

  describe('StatCard 컴포넌트 조건부 색상 검증', () => {
    
    test('증감률(variance) 값이 음수일 경우 텍스트에 text-danger 처리가 된다.', () => {
      // 1. Arrange (가상 DOM 렌더링)
      render(<StatCard title="내일 물량" value="100" variance={-4.5} />);

      // 2. Act (화면에 뜬 텍스트 노드 탐색)
      const varianceTextNode = screen.getByText(/-4.5/i);
      
      // 3. Assert (클래스 소유 여부 단언)
      expect(varianceTextNode).toHaveClass('text-danger');
      expect(varianceTextNode).not.toHaveClass('text-success');
    });

  });
  ```

### 2. Empty State 방어 테스트
- [ ] 차트(`VIEW-006`)나 데이터 표 등에 Props 로 빈 배열 `[]` 이 떨어졌을 때, 화면이 `undefined` 참조 에러를 뿜으며 흰 화면(White screen of death)이 되지 않고, "데이터가 없습니다." 텍스트 노드가 등장하는지 단언(`toBeInTheDocument`)하는 엣지 케이스 테스트 추가.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 예측값의 천 단위 포맷팅 오류 파악**
- Given: 숫자형 15000 (`Number`) 값이 StatCard의 Value Props로 찔림.
- When: 컴포넌트가 렌더됨.
- Then: `screen.getByText('15,000')` 이 성공적으로 Catch 됨으로써 내부 컴포넌트가 `.toLocaleString()` 포맷팅 구문을 망각(Regression)하지 않고 잘 붙여두고 있음을 보장한다.

## ⚙️ Technical Constraints
- API 통신이 들어간 Hook(`useQuery`) 를 사용하는 컴포넌트는 그대로 Testing Library에 올리면 에러가 뿜어지므로, 컴포넌트 UI `Presentational` 과 데이터 `Container` 가 분리된 상태의 순수 UI 컴포넌트만 이 테스트 단상에 올려야 한다 (혹은 Mocking 사용).

## 🏁 Definition of Done (DoD)
- [ ] 음수/양수에 반응하는 DOM ClassName 분기 테스트 통과 구조작성?
- [ ] `toBeInTheDocument` 와 `toHaveClass` 등 DOM 제어 구문 정상 수립?
