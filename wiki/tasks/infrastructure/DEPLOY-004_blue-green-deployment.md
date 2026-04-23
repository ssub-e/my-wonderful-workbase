---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] DEPLOY-004: 가용성 극대화를 위한 '무중단 배포(Blue-Green)' 및 카나리 배포 전략 구성"
labels: 'feature, backend, infra, priority:high'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [DEPLOY-004] 리스크 최소화를 위한 고급 배포 자동화 파이프라인
- 목적: REQ-NF-001 준수. 신규 버전 배포 시 '무중단' 상태를 유지하기 위해 기존 인스턴스(Green)를 유지한 채 신규 인스턴스(Blue)를 띄워 트래픽을 점진적으로 전환하는 Blue-Green 전략 또는 Canary 배포 환경을 구축한다. 이는 배포 도중 발생할 수 있는 서비스 다운타임을 제로(Zero)화 한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라: `DEPLOY-002` (Nginx/Gunicorn), `API-008` (Health Check)
- CI/CD: `ENV-003` (GitHub Actions)

## ✅ Task Breakdown (실행 계획)
- [ ] **Deployment Scripting**:
  - 신규 컨테이너 가동 후 `API-008` 을 통해 헬스 체크가 성공할 때까지 대전환 대기 로직 구현
- [ ] **Load Balancer Routing**:
  - Nginx 또는 Cloud ALB의 타겟 그룹을 교체하는 자동화 스크립트 작성
- [ ] **Database Migration Policy**:
  - 하위 호환성(Backward Compatibility)을 유지하도록 마이그레이션을 선행 수정하고 코드를 나중에 배포하는 2-Step 프로세스 확립
- [ ] **Rollback Automation**:
  - 전환 직후 오류 급증 시(`MONITOR-001` 연계) 자동으로 이전 버전 타겟으로 롤백하는 로직 트리거
- [ ] **Canary Logic (Optional)**: 전체 트래픽의 10%만 신규 버전으로 우선 할당하는 Nginx `split_clients` 설정 구성

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 무중단 배포 성공
- Given: 버전 1.0이 구동 중인 상태
- When: 버전 1.1 배포 버튼 클릭
- Then: 1.1이 완전히 뜰 때까지 1.0이 계속 응답하다가, 헬스 체크 성공 시점에 트래픽이 1.1로 유지 및 전환되며 유저는 다운타임을 인지하지 못해야 한다.

Scenario 2: 배포 중 장애 감지 및 롤백
- Given: 버전 1.1 배포 직후 500 에러 발생률이 5%를 초과함
- When: 모니터링 시스템 감색
- Then: 배포 파이프라인이 즉시 중단되고, 로드 밸런서는 살아있는 1.0 인스턴스로 모든 트래픽을 긴급 회귀시켜야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 배포 전환 시 세션 단절 방지를 위해 Redis 기반의 중앙 집중형 세션(`INFRA-002`) 필수 활용
- 인프라: Blue/Green 환경을 위한 일시적인 여유 서버 리소스 비용 고려
- 보안: 배포 가동 중 관리자 이외의 접근 차단을 위한 Maintenance Mode 수동 제어 기능 포함

## 💻 Implementation Snippet (Nginx Switch Idea)
```bash
# deployment/switch_blue_green.sh
NEW_TARGET=$1 # "blue" or "green"
sed -i "s/upstream backend { server .*/upstream backend { server localhost:$PORT; }/" /etc/nginx/conf.d/api.conf
nginx -s reload
echo "Switched traffic to $NEW_TARGET"
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 수동 배포 테스트 시 `top` 또는 모니터링 툴상에서 서비스 중단 없이 가동이 전환되었는가?
- [ ] 배포 이력이 `AUDIT_LOG` 에 정확히 기록되는가?

## 🚧 Dependencies & Blockers
- Depends on: `API-008`, `DEPLOY-001`
- Blocks: 없음
