---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-005: 외부 시스템 연동용 데이터 인바운드(Inbound) 웹훅 API"
labels: 'feature, backend, api, priority:high'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-005] 범용 데이터 수집을 위한 인바운드 웹훅(Push Model) API
- 목적: REQ-FUNC-014 확장. 카페24 등 기 구축된 어댑터 외에, 고객사가 자체 개발한 ERP나 WMS에서 우리 시스템으로 직접 실시간 주문/재고 데이터를 밀어넣을 수(Push) 있도록 표준화된 REST 엔드포인트를 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 보안 인증: `API-001` (M2M API-Key 인증)
- 데이터 검증: `DB-003`(Order), `DB-004`(Inventory) 스키마

## ✅ Task Breakdown (실행 계획)
- [ ] `app/api/v1/endpoints/webhooks.py` 신설
- [ ] **Endpoint 1: 주문 데이터 푸시** (`POST /webhook/orders`)
  - 단건 및 리스트 동시 지원
  - 스키마 변환 레이어: 외부 포맷 -> 시스템 내부 `OrderCreateDto` 매핑
- [ ] **Endpoint 2: 재고 데이터 푸시** (`POST /webhook/inventory`)
- [ ] 비동기 처리: 웹훅 요청 수신 즉시 202 Accepted 반환 후, 백그라운드 태스크에서 DB 저장 및 이상치 검사(`DQ-001`) 수행
- [ ] 웹훅 전용 감사 로그(`WEBHOOK_LOG`): 요청 Origin IP, Payload 요약, 처리 결과 저장

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 외부 ERP의 주문 데이터 수신 성공
- Given: 유효한 `X-API-Key`를 가진 외부 서버
- When: JSON 바디에 50건의 주문 데이터를 담아 `POST /webhook/orders` 호출
- Then: 서버는 `202 Accepted`를 즉시 반환하며, 1분 이내에 `ORDER` 테이블에 50건의 데이터가 적재된다.

Scenario 2: 잘못된 데이터 구조 전송 시 반려
- Given: 필수 필드(`platform_order_id`)가 누락된 Payload
- When: 웹훅 호출
- Then: 서버는 `422 Unprocessable Entity`와 함께 어느 필드가 누락되었는지 상세 에러를 반환한다.

## ⚙️ Technical & Non-Functional Constraints
- 보안: `tenant_id`는 API Key에 종속되므로 페이로드에 포함하지 않고 서버 내부에서 매핑
- 가용성: 웹훅 폭주에 대비하여 `Inbound Rate Limit`을 테넌트당 100 req/sec 로 설정
- 유연성: 향후 지원되지 않는 필드는 `extra_data` (JSONB) 컬럼에 보존

## 💻 Implementation Snippet (Webhook Router)
```python
@router.post("/orders", status_code=202)
async def inbound_orders_webhook(
    *,
    db: AsyncSession = Depends(get_db),
    payload: List[ExternalOrderDto],
    background_tasks: BackgroundTasks,
    api_key_tenant: Tenant = Depends(get_tenant_from_api_key)
):
    """
    외부 시스템으로부터 주문 데이터를 Push 받아 비동기로 처리
    """
    # 1. 수신 로그 기록
    await log_webhook_request(db, api_key_tenant.id, "ORDERS", len(payload))
    
    # 2. 비동기 워커에 적재 및 검증 위임
    background_tasks.add_task(
        process_inbound_data,
        tenant_id=api_key_tenant.id,
        data_type="ORDER",
        payload=payload
    )
    
    return {"message": "Data received and queued for processing", "count": len(payload)}
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] API Key 인증 실패 시 `401 Unauthorized`가 정확히 반환되는가?
- [ ] 대량 데이터 수집 시 DB 커넥션 풀 부족 현상이 발생하지 않는가?

## 🚧 Dependencies & Blockers
- Depends on: `API-001`, `DB-003`
- Blocks: 없음
