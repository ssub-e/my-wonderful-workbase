---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-013: 시스템 감사 프레임(Audit Log) 페이징 데이터 그리드"
labels: 'frontend, view, admin, phase:5, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-013] 보안 감사 로그 열람용 페이지네이션 형태의 테이블 뷰
- 목적: B2B 엔터프라이즈의 규제 요건인 "누가 언제 무엇을 했는지"(`REQ-NF-021`)를 화주사 관리자가 화면에서 이력으로 볼 수 있도록 구성한다. `ADMIN-006` API의 오프셋 기반 페이징 모델을 React 컴포넌트로 렌더링한다.

## 🔗 References (Spec & Context)
- 연결 API: `ADMIN-006` (GET `/admin/audit-logs`)
- 디자인 컴포넌트: `VIEW-002` (Card, Badge 재사용)

## ✅ Task Breakdown (실행 계획)

### 1. Paginated Query Hook 설계
- [ ] `src/hooks/useAuditLogs.ts`:
  ```typescript
  import { useQuery } from '@tanstack/react-query';
  import { apiClient } from '@/api/client';
  import { useState } from 'react';

  export const useAuditLogs = () => {
    const [page, setPage] = useState(1);
    const size = 20;

    const query = useQuery({
      queryKey: ['audit-logs', page],
      queryFn: async () => {
        const { data } = await apiClient.get(`/admin/audit-logs?page=${page}&size=${size}`);
        return data; // { data: [], total_count, page } 형태
      },
      placeholderData: (prev) => prev, // 페이지 넘어갈 때 깜빡임 방지용 캐시 유지
    });

    return { ...query, page, setPage, size };
  };
  ```

### 2. 테이블 UI 및 JSON 데이터 파싱 포맷터
- [ ] `src/pages/settings/AuditLogView.tsx`:
  - `data.data.map()` 을 통해 표의 `<tr/>` 을 직조한다.
  - 액션 타입(`UPDATE`, `CREATE`)에 따라 `Badge` 색상을 다르게 렌더링한다.
  - JSON 변경 내역인 `changes` 객체를 사람이 읽을 수 있도록 `JSON.stringify(..., null, 2)` 형태의 툴팁 혹은 펼침 메뉴 안의 `<pre className="text-xs">` 태그로 밀어 넣는다.
  - 하단에 `[<] 1 2 3 4 5 [>]` 형태의 숫자로 구성된 Pagination Footer 컴포넌트를 배치하고 `setPage(clickedNum)` 을 onClick에 바인딩한다.

## 🧪 Acceptance 초도 기준 (BDD/GWT)

**Scenario 1: PlaceholderData를 통한 깜빡임(Flicker) 없는 페이징 UX**
- Given: 사용자가 1페이지에서 표를 보다가 "2(다음)" 버튼을 누름.
- When: `useQuery` 의 page 값이 스왑되어 네트워크 요청이 날아감.
- Then: `placeholderData: keepPreviousData` 가 적용되어 있으므로 1페이지 표가 하얗게(로딩 스피너) 날아가지 않고 그대로 유지되다가 데이터 도착 즉시 2페이지 내용으로 스위아웃 되어 눈의 피로를 막는 고급 SPA 경험을 제공한다.

## ⚙️ Technical Constraints
- JSON `changes` 객체 내부 키값이 클라이언트 브라우저의 DOM 계층 제한을 넘을 수 있는 매우 긴 텍스트(예: LLM 프롬프트 원문)를 담고 있다면, 일정 글자수 이상에서 `...더보기` 로 말줄임표(Truncate) 처리하는 커스텀 로직을 짜야 UI 붕괴를 예방할 수 있다.

## 🏁 Definition of Done (DoD)
- [ ] 페이지네이션 상태(`useState(page)`) 관리가 포함된 React Query 훅 작성?
- [ ] `changes` 필드의 형태(Object)를 견고하게 핸들링 하는 UI 셀 구현 확인?
