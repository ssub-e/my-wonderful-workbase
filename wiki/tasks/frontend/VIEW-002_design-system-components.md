---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-002: 시스템 디자인 공통 (Core) UI 컴포넌트 셋업"
labels: 'frontend, view, ui, phase:5, priority:high'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-002] 공용 디자인 컴포넌트(Design System) 구현
- 목적: B2B 대시보드 내에서 수십 번 재사용 될 버튼(Button), 통계 카드(Card), 폼 인풋(Input) 등의 단위 컴포넌트를 미리 개발하여 일관된 UX/UI 룩앤필(Look-and-Feel)을 보장한다 (현대적인 Shadcn UI 스타일 지향).

## 🔗 References (Spec & Context)
- 테마 명세: `VIEW-001` 에서 잡힌 Tailwind Colors (`primary`, `surface` 등)

## ✅ Task Breakdown (실행 계획)

### 1. Button 컴포넌트
- [ ] `src/components/ui/Button.tsx`:
  ```tsx
  import React from 'react';
  import { cn } from '@/utils/cn';

  interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
    variant?: 'primary' | 'outline' | 'danger';
    size?: 'sm' | 'md' | 'lg';
  }

  export const Button: React.FC<ButtonProps> = ({ variant = 'primary', size = 'md', className, ...props }) => {
    const baseClass = "inline-flex items-center justify-center rounded-md font-medium transition-colors focus:ring-2 focus:outline-none disabled:opacity-50 disabled:pointer-events-none";
    const variantClass = {
      primary: "bg-primary text-white hover:bg-blue-800",
      outline: "border border-gray-300 text-gray-700 hover:bg-gray-50",
      danger: "bg-danger text-white hover:bg-red-600"
    }[variant];
    const sizeClass = {
      sm: "h-8 px-3 text-sm",
      md: "h-10 px-4 py-2",
      lg: "h-12 px-6 text-lg"
    }[size];

    return (
      <button className={cn(baseClass, variantClass, sizeClass, className)} {...props} />
    );
  };
  ```

### 2. Card 레이아웃 컴포넌트
- [ ] `src/components/ui/Card.tsx`: 대시보드 위젯들을 담을 하얀색 빈 상자.
  ```tsx
  export const Card = ({ children, className }: { children: React.ReactNode, className?: string }) => (
    <div className={cn("bg-surface rounded-xl border border-gray-200 shadow-sm overflow-hidden", className)}>
      {children}
    </div>
  );
  // CardHeader, CardContent 등 서브 컴포넌트로 분리 조합 권장
  ```

### 3. Badge (상태 뱃지) 컴포넌트
- [ ] `src/components/ui/Badge.tsx`: "배송중", "완료", "처리중" 등 상태를 나타내는 미니 알약 뱃지. (Success/Danger 색상 분기).

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 동적 클래스 병합(tailwind-merge) 동작**
- Given: 개발자가 페이지에서 `<Button variant="primary" className="mt-4 bg-red-500" />` 라고 선언함
- When: 내부 `cn()` 유틸이 동작함
- Then: Tailwind 로직의 충돌(primary의 파란색 vs 밖에서 준 붉은색)이 발생했을 때, `tailwind-merge` 규칙에 기반해 제일 마지막에 주입한 `bg-red-500`으로 안전하게 오버라이딩 오버라이드 되어 브라우저에 나타난다.

## ⚙️ Technical Constraints
- 순수 CSS 분리를 막기 위해 모든 스타일링은 Tailwind 클래스 네임스페이스 안에서만 해결해야 한다 (인라인 style 사용 지양).
- 컴포넌트 인터페이스(`Props`)는 `JSX.IntrinsicElements` 를 확장하여 표준 HTML 어트리뷰트(onClick, disabled 등)를 무조건 상속받게 짜야 한다.

## 🏁 Definition of Done (DoD)
- [ ] Button (3종 Variant), Card 컴포넌트 생성?
- [ ] `cn()` 유틸(`clsx` + `twMerge`) 클래스 병합 지원 완성?
