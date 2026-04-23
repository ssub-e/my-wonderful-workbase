---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-006: 외부 분석용 대용량 '벌크 데이터 익스포트(Export)' API"
labels: 'feature, backend, api, priority:medium'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-006] 전사 데이터 활용을 위한 대용량 CSV/JSON 추출 게이트웨이
- 목적: 화주사의 데이터 분석팀이 Tableau, PowerBI 또는 사내 데이터 웨어하우스(DW)에서 우리 시스템의 예측-실적 데이터를 분석에 활용할 수 있도록, 수십만 건의 레코드를 안정적으로 추출해 주는 전용 API를 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 관련 태스크: `FILE-001` (결과물 S3 업로드), `API-001` (M2M Key)
- 성능 제약: `INFRA-006` (Read-Replica 사용 필수)

## ✅ Task Breakdown (실행 계획)
- [ ] 익스포트 전용 라우터 `POST /api/v1/export/forecast-results` 구현
- [ ] **Background Export Pipeline**:
  - 요청 즉시 `export_id` 반환 후 비동기 작업 시작
  - Read-Replica DB에서 데이터를 스트리밍(`AsyncGenerator`) 방식으로 읽어옴
  - 메모리 부하 방지를 위해 CSV 파일로 실시간 쓰기 및 압축
- [ ] 완료 시 유저에게 `Event-Stream`(`API-002`) 또는 이메일 알림 송신
- [ ] 익스포트 이력 및 다운로드 URL(Presigned) 유효기간 관리
- [ ] **Feature Gating**: 구독 플랜(`BILLING-002`)에 따라 추출 가능한 데이터 기간(예: Basic 1년, Pro 3년) 제한

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 1년치 데이터 벌크 추출 시나리오
- Given: 50만 건의 데이터가 축적된 Pro 요금제 테넌트
- When: 1년치 데이터 추출 요청(Format: CSV) 호출
- Then: 서버는 `202 Accepted`를 반환하며, 잠시 후 "추출이 완료되었습니다" 알림과 함께 50MB 상당의 압축 파일 다운로드 링크가 제공된다.

Scenario 2: 추출 중 권한 및 기간 필터 검증
- Given: Basic 요금제 유저 (3년치 추출 시도)
- When: 추출 요청 호출
- Then: "현재 플랜으로는 최대 1년치 데이터만 추출 가능합니다" 라는 에러 메시지가 반환되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 마스터 DB 부하를 막기 위해 반드시 `INFRA-006` 읽기 복제본 커넥션을 사용
- 안정성: 동시 추출 요청 수를 테넌트당 1개로 제한(Queueing)하여 서버 자원 고갈 방지
- 보안: 추출된 파일명에 테넌트 ID를 포함한 난독화 적용 및 24시간 후 자동 삭제

## 💻 Implementation Snippet (Streaming Export Logic)
```python
import csv
import io

async def generate_csv_stream(query_result_generator):
    """
    DB 결과를 메모리에 다 올리지 않고 CSV 스트림으로 변환
    """
    output = io.StringIO()
    writer = csv.writer(output)
    
    # Header
    writer.writerow(["date", "sku_id", "forecast_qty", "actual_qty", "error_rate"])
    
    async for row in query_result_generator:
        writer.writerow([row.date, row.sku_id, row.forecast_qty, row.actual_qty, row.error_rate])
        if output.tell() > 1024 * 1024:  # 1MB 마다 yield
            yield output.getvalue()
            output.seek(0)
            output.truncate(0)
    
    yield output.getvalue()
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 10만 건 이상의 데이터로 추출 시 메모리 누수가 없는지 확인 완료?
- [ ] `API-001` 의 API-Key로 인증이 가능한가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-006`, `FILE-001`, `BILLING-002`
- Blocks: 없음
