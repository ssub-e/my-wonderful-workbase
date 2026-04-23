---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INFRA-008: 비즈니스 연속성을 위한 DB '특정 시점 복구(PITR)' 및 스냅샷 자동화"
labels: 'feature, backend, infra, priority:high'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-008] 재해 복구(DR)를 위한 DB 스냅샷 및 시점 복구 인프라
- 목적: 실수로 인한 대량 데이터 삭제나 DB 오염 사고 발생 시, 데이터를 1초 전 상태로 되돌릴 수 있는 특정 시점 복구(Point-in-Time Recovery) 환경을 구축한다. 이는 B2B SaaS 환경에서 고객사의 소중한 실적 데이터를 철벽 보호하기 위한 최후의 방어선이다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라: AWS RDS PostgreSQL 또는 Supabase 기반 장기 보존 정책
- 관련 태스크: `INFRA-005` (S3 동기화)

## ✅ Task Breakdown (실행 계획)
- [ ] **Automated Snapshot**: 매일 새벽 04:00에 전체 DB 스냅샷을 생성하고 30일간 보존하도록 설정
- [ ] **WAL(Write-Ahead Log) Archiving**: 연속적인 복구를 가능케 하는 WAL 로그를 S3에 실시간 아카이빙 하는 로직 구성
- [ ] **Recovery Drill Script**: 
  - 관리자가 특정 타겟 시간(`restore_time`)을 입력하면 해당 시점의 새 DB 인스턴스를 생성하는 자동화 스크립트 작성
- [ ] **Multi-AZ Failover**: 가용 영역(AZ) 장애 시 즉시 복제본으로 전환(Failover) 되는 클러스터 구성 검증
- [ ] 모니터링: 백업 실패 시 Sentry 및 Slack(`MONITOR-001`)으로 즉시 경보 발송

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 긴급 데이터 복구 시나리오 (PITR)
- Given: 오늘 오후 2시 15분에 관리자의 실수로 `ORDER` 테이블 1만 건이 증발함
- When: 오후 2시 14분 상태로 PITR 복구를 실행함
- Then: 약 n분 이내에 2시 14분 시점의 데이터가 온전히 복구된 새로운 DB 인스턴스가 Ready 상태가 되어야 한다.

Scenario 2: 백업 데이터 보존 기간 검증
- Given: 스냅샷 설정이 완료된 운영 서버
- When: 31일이 경과함
- Then: 스토리지 비용 최적화를 위해 30일이 넘은 가장 오래된 스냅샷은 자동으로 삭제되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 백업 및 스냅샷 생성 중 운영 DB의 I/O 지연(Latency)이 10% 미만으로 유지되어야 함
- 보안: 모든 백업 데이터는 `FILE-001` 정책에 따라 암호화된 상태로 S3에 저장되어야 하며, 외부 접근은 철저히 차단됨
- 신뢰성: 분기당 1회 '무중단 복구 훈련'을 수행할 수 있는 툴링 제공

## 💻 Implementation Snippet (AWS CLI Backup Config Idea)
```bash
# AWS RDS Snapshot retention 설정 예시
aws rds modify-db-instance \
    --db-instance-identifier srs-prd-db \
    --backup-retention-period 30 \
    --preferred-backup-window 04:00-05:00 \
    --apply-immediately

# 특정 시점 복구 명령 예시
aws rds restore-db-instance-to-point-in-time \
    --target-db-instance-identifier srs-prd-db-restored \
    --source-db-instance-identifier srs-prd-db \
    --restore-time 2026-04-22T14:14:00Z
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 실제 복구 테스트(Dry-run)를 통해 데이터 재구축 시간이 측정되었는가?
- [ ] `RTO`(복구 목표 시간) 4시간 이내를 달성하는 구성인가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-001`
- Blocks: 없음
