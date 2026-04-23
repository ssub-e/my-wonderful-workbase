---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DASH-012: 글로벌 시스템 현황 모니터링을 위한 '슈퍼 어드민(Super Admin) 통합 대시보드'"
labels: 'feature, frontend, admin, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DASH-012] 본사 운영팀용 전사 지표 통합 관제 뷰
- 목적: 개별 화주사가 아닌, SaaS 본사 운영진이 전체 테넌트의 활동량, 시스템 리소스 사용량, 결제 현황, 주요 오류 발생 건수를 한눈에 파악하여 선제적으로 대응할 수 있는 마스터 대시보드를 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 사용자 권한: `SUPER_ADMIN` 전용 (`AUTH-001`)
- 관련 태스크: `SUPER-001` (테넌트 관리), `MONITOR-001` (오류 알림)

## ✅ Task Breakdown (실행 계획)
- [ ] **Global Stats Aggregator API**:
  - 총 테넌트 수, 활성 유저 수, 오늘 총 주문 처리량, AI 추론 성공률 등 전역 지표 산출
- [ ] **Tenant Activity Map**: 최근 24시간 내 활동이 가장 많은 상위 10개 테넌트 리스트 렌더링
- [ ] **System Resource Widget**: 서버(CPU/MEM), DB(Storage), Redis(Memory) 사용 현황 인디케이터 연동
- [ ] **Billing Trend Chart**: 이번 달 예상 매출 및 신규 구독/해지 추이 시각화
- [ ] **Incident Timeline**: 전사적으로 발생한 High-priority 에러(`MONITOR-001`)의 타임라인 노출 및 조치 상태 시각화

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 슈퍼 어드민 대시보드 접근 제어
- Given: 일반 화주사(MD) 계정으로 로그인한 상태
- When: `/super-admin/dashboard` URL로 직접 접근 시도
- Then: `403 Forbidden` 과 함께 접근이 거부되어야 하며, 해당 메뉴가 UI상에서도 노출되지 않아야 한다.

Scenario 2: 실시간 전사 데이터 요약 확인
- Given: 슈퍼 어드민 권한으로 로그인
- When: 대시보드 진입
- Then: 특정 테넌트가 아닌, 모든 화주사의 통합 주문량 합계와 시스템의 전체 CPU 사용률이 로딩 스켈레톤 UI를 거쳐 정확히 표시되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 전사 집계 쿼리는 매우 무겁기 때문에 실시간이 아닌 5분 단위의 'Materialized View' 또는 Redis 캐시 사용 권장
- 보안: 슈퍼 어드민 대시보드 데이터는 어떠한 경우에도 외부(화주사)로 유출되지 않도록 라우터 레벨에서 Role 철저 검증
- UX: '위험 수치(예: 에러 급증)' 발생 시 대시보드 UI 전체에 붉은색 경고 오버레이 및 사운드 온오프 기능 제공

## 💻 Implementation Snippet (Super Admin Stats API Idea)
```python
@router.get("/global-stats", dependencies=[Depends(RoleChecker([UserRole.SUPER_ADMIN]))])
async def get_global_stats(db: AsyncSession = Depends(get_db)):
    """
    SaaS 본사용 전사 통계 지표 반환 (캐싱 적용 필수)
    """
    cache_key = "super_admin:global_stats"
    if cached_data := await redis.get(cache_key):
        return json.loads(cached_data)

    stats = {
        "total_tenants": await db.scalar(select(func.count(Tenant.id))),
        "active_users_24h": await get_active_users_count(db),
        "total_revenue_mtd": await get_monthly_revenue(db),
        "system_health": await get_infra_status()
    }
    
    await redis.setex(cache_key, 300, json.dumps(stats)) # 5분 캐싱
    return stats
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 집계 쿼리가 DB 마스터에 부하를 주지 않도록 Read-Replica(`INFRA-006`)를 사용했는가?
- [ ] 슈퍼 어드민 전용 IP 화이트리스트 기능과 연동되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-006`, `SUPER-001`
- Blocks: 없음
