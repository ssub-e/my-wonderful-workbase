---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] INFRA-007: 장기 감사 로그 아카이빙 및 S3 Glacier 자동 이관 파이프라인"
labels: 'feature, backend, infra, priority:low'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [INFRA-007] 고비용 스토리지 절감을 위한 로그 콜드 스토리지 이관 정책
- 목적: REQ-NF-010 준수. 수년간 쌓이는 `AUDIT_LOG` 및 `WEBHOOK_LOG` 는 DB 용량의 상당 부분을 차지하며 쿼리 성능을 저하시킨다. 이를 해결하기 위해 6개월이 지난 로그는 DB에서 삭제하고, 조회 빈도가 낮은 S3 Glacier(Cold Storage)로 압축 및 암호화하여 아카이빙하는 자동화 파이프라인을 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 데이터 보존 기간 정책: 최소 3년 (법적 규제 대비)
- 관련 태스크: `FILE-001` (S3 어댑터), `DB-010` (감사 로그 스키마)

## ✅ Task Breakdown (실행 계획)
- [ ] `LOG_ARCHIVE_HISTORY` 테이블 설계: id, archive_date, log_type, record_count, s3_path, checksum
- [ ] **LogExporterWorker** 구현:
  - 180일 이전의 로그 데이터를 JSON/CSV 형태로 추출
  - 추출된 파일을 Gzip 으로 압축하여 용량 절감
  - `FILE-001` 을 통해 S3 `archive/` 경로로 업로드
- [ ] **Purge Logic**: S3 업로드가 성공(Checksum 검증 완료)한 데이터에 한해 DB 원본 테이블에서 `DELETE` 수행
- [ ] **Glacier Transition Rule**: S3 버킷 정책(Lifecycle Policy)을 설정하여 `archive/` 아래의 파일은 업로드 30일 후 Glacier 로 자동 전환
- [ ] 필요시 아카이빙된 로그를 다시 불러오는(Restore) 수동 복구 스크립트 작성

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 정기 로그 아카이빙 테스트
- Given: DB에 200일 전의 감사 로그 10만 건이 존재함
- When: 아카이브 워커가 실행됨
- Then: 로그 10만 건이 파일로 압축되어 S3에 저장되고, DB에서는 해당 기간의 로그가 삭제되어 디스크 용량이 확보된다.

Scenario 2: 데이터 정합성 검증 (Safety First)
- Given: 아카이브 작업 중 S3 업로드 네트워크 장애 발생
- When: 업로드 시도 중 에러 반환
- Then: DB의 로그는 삭제되지 않아야 하며(Rollback), 장애 상황이 운영팀에 알림 전송되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 안정성: 대량 삭제(Delete) 작업 시 DB 락(Lock) 및 트랜잭션 로그 비대화 방지를 위해 1,000건 단위로 쪼개서 삭제
- 보안: 아카이브된 파일은 `SEC-001` 정책에 따라 서버 사이드 암호화(SSE-S3) 필수 적용
- 가용성: 아카이브 워커는 시스템 부하가 가장 적은 새벽 시간대(AM 03:00)에만 구동

## 💻 Implementation Snippet (Archive Policy Idea)
```python
async def archive_old_logs(db: AsyncSession, log_table: Any, days: int = 180):
    cutoff_date = datetime.utcnow() - timedelta(days=days)
    
    # 1. 대상 데이터 스트리밍 조회
    async with db.stream(select(log_table).where(log_table.created_at < cutoff_date)) as result:
        # 파일 생성 및 업로드 로직 (Streaming to S3)
        s3_path = await upload_stream_to_s3(result)
        
    # 2. 성공 시에만 삭제
    if s3_path:
        await db.execute(delete(log_table).where(log_table.created_at < cutoff_date))
        await db.commit()
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] S3 수명 주기 정책(Lifecycle) 설정이 Terraform/콘솔에 반영되었는가?
- [ ] 아카이브 실행 결과가 `LOG_ARCHIVE_HISTORY` 에 정확히 기록되는가?

## 🚧 Dependencies & Blockers
- Depends on: `FILE-001`, `DB-010`
- Blocks: 없음
