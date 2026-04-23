---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DASH-011: 화주사용 사용자 계정 관리 및 그룹 지정 어드민 뷰"
labels: 'feature, frontend, admin, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-011] 화주사 사내 직원 계정 통합 관리 센터
- 목적: `ADMIN-010` 에서 구축한 유저 그룹 및 권한 매트릭스를 기반으로, 화주사 관리자가 사내 직원을 초대하고, 프로필을 수정하며, 특정 부서 그룹(예: '인천 센터팀')에 배정하거나 권한을 회수할 수 있는 통합 관리자 화면을 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 백엔드 의존성: `ADMIN-010`, `ADMIN-004` (초대 API)
- UI 컴포넌트: `VIEW-012` (초대 모달), `VIEW-018` (그리드)

## ✅ Task Breakdown (실행 계획)
- [ ] **User Management Grid**: 사내 전체 사용자 목록 (이름, 이메일, 그룹, 가입일, 상태) 렌더링
- [ ] **Group Assignment UI**: 특정 유저 선택 시 아바타 옆 드롭다운을 통해 소속 그룹을 즉시 변경하는 인라인 액션
- [ ] **Invitation Link Generator**: 초대 이메일을 놓친 유저를 위해 초대 전용 링크를 복사하여 전달하는 기능
- [ ] **Status Control**: 퇴사자 또는 휴직자를 위한 '계정 비활성화(Suspend)' 토글 스위치 구현
- [ ] `react-query`의 `mutation`을 사용하여 권한 변경 즉시 반영 및 UI 캐시 갱신

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 유저 소속 그룹 변경
- Given: '홍길동' 유저가 '일반 직원' 그룹에 있음
- When: 관리자가 대시보드 관리 탭에서 홍길동의 그룹을 '팀장'으로 변경함
- Then: 즉시 API가 호출되어 DB가 업데이트되고, 홍길동 유저의 다음 요청부터 팀장 권한(`ADMIN-010`)이 적용된다.

Scenario 2: 보안 사고 대응 계정 정지
- Given: 보안 이슈가 발생한 특정 유저 계정
- When: 관리자가 '계정 활성' 스위치를 OFF로 변경함
- Then: 해당 유저의 현재 JWT 세션은 무효화(Redis Blacklist) 처리되어야 하며, 즉시 로그아웃 상태가 되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: '현재 나 자신(관리자)'의 권한을 스스로 박탈하거나 정지하지 못하도록 방어 로직 적용
- 보안: 그룹 변경 등 민감 작업은 반드시 `AUDIT_LOG` 에 'ADMIN_CHANGE_USER_ROLE' 타입으로 기록
- 반응형: 모바일 화면에서는 그리드 대신 리스트 카드 형태로 자동 변환(`VIEW-026` 연계)

## 💻 Implementation Snippet (User List Action Logic)
```tsx
const UserGroupSwitcher = ({ userId, currentGroupId, groups }: Props) => {
  const mutation = useMutation({
    mutationFn: (newGroupId: string) => api.patch(`/admin/users/${userId}/group`, { group_id: newGroupId }),
    onSuccess: () => {
      toast.success("사용자 그룹이 변경되었습니다.");
      queryClient.invalidateQueries(["users"]);
    }
  });

  return (
    <Select value={currentGroupId} onValueChange={(val) => mutation.mutate(val)}>
      <SelectTrigger className="w-[180px]">
        <SelectValue placeholder="Select Group" />
      </SelectTrigger>
      <SelectContent>
        {groups.map(g => <SelectItem key={g.id} value={g.id}>{g.name}</SelectItem>)}
      </SelectContent>
    </Select>
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 그룹 변경 시 해당 유저의 권한이 즉시 동기화되는가?
- [ ] 어드민 계정 자신에 대한 '삭제/비활성화' 버튼이 비활성화되는가?

## 🚧 Dependencies & Blockers
- Depends on: `ADMIN-010`, `VIEW-018`
- Blocks: 없음
