---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ADMIN-010: 엔터프라이즈용 '부서별 권한 매트릭스(Role Matrix)' 및 유저 그룹 관리"
labels: 'feature, backend, admin, priority:medium'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADMIN-010] 고도화된 RBAC 권한 관리 및 유저 그룹링 시스템
- 목적: 화주사의 규모가 커짐에 따라(예: 본사, 지역 센터 A, 지역 센터 B), 유저별로 권한을 하나하나 부여하는 것은 비효율적이다. 따라서 '본사 MD 그룹', '현장 관리자 그룹' 등 유저 그룹을 먼저 정의하고, 각 그룹별로 메뉴/기능 접근 권한을 매트릭스 형태로 매핑하여 대규모 인원을 효율적으로 관리한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 기본 권한 시스템: `AUTH-001` (RBAC)
- 요구사항: REQ-FUNC-011 (사용자별 권한 차등화)

## ✅ Task Breakdown (실행 계획)
- [ ] `USER_GROUP` 테이블 설계: id, tenant_id, name, description
- [ ] `GROUP_PERMISSION` 테이블 설계: group_id, feature_key(예: DASHBOARD_VIEW, ADMIN_EDIT), permission_level(NONE/READ/WRITE)
- [ ] **Permission Resolver** 고도화:
  - 현재 유저의 소속 그룹을 찾고, 해당 그룹에 부여된 `feature_key` 권한을 합산하여 최종 권한 도출
- [ ] 권한 매트릭스 UI용 API 구축: `GET /api/v1/admin/permissions/matrix` (기능 리스트와 현재 그룹별 체크 여부 반환)
- [ ] 그룹 내 유저 일괄 추가/삭제(Bulk Update) 기능 구현
- [ ] `DASHHBOARD` 접근 시 해당 유저의 그룹 권한에 따라 특정 탭(예: 백오피스) 자동 비활성화 로직

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 그룹 권한 변경 시 즉시 반영
- Given: '현장 센터원' 그룹에 '대시보드 조회' 권한만 있고 '설정 수정' 권한은 없음
- When: 관리자가 해당 그룹에 '설정 수정' 권한을 앱 매트릭스에서 체크하고 저장함
- Then: 해당 그룹에 속한 모든 유저는 즉시 설정 페이지에 접근할 수 있게 됨.

Scenario 2: 유저 그룹 이동 테스트
- Given: 'MD' 그룹 유저가 '본사 임원' 그룹으로 이동함
- When: 해당 유저가 리포트 승인 버튼을 클릭함
- Then: 이전에 거부되던 상위 권한 작업이 '본사 임원' 권한을 상속받아 정상 수행됨.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 매 API 호출 시 권한 체킹 부하를 줄이기 위해 유저 권한 정보를 세션/JWT 혹은 Redis에 캐싱
- 신뢰성: `SUPER_ADMIN` 권한의 경우 그룹 설정과 관계없이 전 기능을 무조건 통과하는 'Master Key' 로직 유지
- 확장성: 향후 커스텀 권한(Custom Role) 정의 기능을 위해 하드코딩된 Role 대신 DB 기반 Permission Key 체계 유지

## 💻 Implementation Snippet (Permission Matrix Schema)
```python
class PermissionLevel(str, Enum):
    NONE = "none"
    READ = "read"
    WRITE = "write"

class GroupPermission(SQLModel, table=True):
    group_id: uuid.UUID = Field(foreign_key="user_group.id", primary_key=True)
    feature_key: str = Field(primary_key=True) # 예: "BILLING_ACCESS", "DASHBOARD_STATS"
    level: PermissionLevel = Field(default=PermissionLevel.NONE)
    
    # Audit trail
    updated_at: datetime = Field(default_factory=datetime.utcnow)
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 권한 매트릭스의 모든 조회/수정 API가 헥사고날 구조 내에서 유닛 테스트를 통과했는가?
- [ ] 프론트엔드가 권한이 없는 버튼을 `Disabled` 처리할 때 이 API 결과를 참조하는가?

## 🚧 Dependencies & Blockers
- Depends on: `AUTH-001`, `ADMIN-012`
- Blocks: 없음
