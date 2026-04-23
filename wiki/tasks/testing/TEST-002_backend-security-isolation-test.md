---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Test] TEST-002: 보안 (테넌트 데이터 교차 노출 제한) 캡슐화 검증 테스트"
labels: 'test, qa, security, phase:7, priority:critical'
assignees: ''
type: task
tags: [task, testing]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [TEST-002] Multi-Tenant Row-Level 격리(`DB-011`) 무결성 테스트
- 목적: B2B SaaS의 가장 치명적인 사고인 'A 회사가 B 회사의 영업 데이터를 조회하는 현상' 이 절대 불가능함을 `pytest` 로 가혹하게 증명해낸다.

## 🔗 References (Spec & Context)
- 검증 대상 모듈: `DB-011` (TenantAwareRepository)
- 인증 연계망: `AUTH-001` (JWT 페이로드 기반 User Context)

## ✅ Task Breakdown (실행 계획)

### 1. 교차 접근 및 침해 시도 시나리오 테스트 작성
- [ ] `tests/domain/test_tenant_isolation.py`:
  ```python
  import pytest
  from app.domain.order.repository import OrderRepository
  from app.domain.order.model import Order

  def test_cross_tenant_access_denial(session):
      # 1. 인위적으로 A회사와 B회사 데이터를 섞어 넣음
      session.add(Order(id=1, tenant_id="TENANT_A", predicted_qty=500))
      session.add(Order(id=2, tenant_id="TENANT_B", predicted_qty=300))
      session.commit()

      # 2. 레파지토리 미들웨어를 "TENANT_A" 의 컨텍스트로 한정하여 인스턴스화 수행
      repo_for_a = OrderRepository(session, current_tenant_id="TENANT_A")

      # 3. 침해 시도 동작 (전체 목록 내놔!)
      results = repo_for_a.get_all()

      # 4. 검증 (Assertion)
      assert len(results) == 1
      assert results[0].tenant_id == "TENANT_A"
      
      # 5. 침해 시도 동작 (ID 2번 B회사껄 내놔!)
      b_data = repo_for_a.get_by_id(2)
      assert b_data is None  # 절대 반환되어선 안됨!
  ```

### 2. 강제 위변조(SSRF) 시도에 대한 차단 
- [ ] "TENANT_A" 컨텍스트를 가진 레파지토리가 `insert/update` 할 때, Payload 객체에 억지로 `tenant_id="TENANT_B"` 를 찌르고 저장 시도할 시 이를 거부 혹은 자체 Override 쳐버리는 안전장치 작동 테스트를 추가한다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: B회사 PK 직접 조회 해킹 시도 방어**
- Given: A회사의 악의적 담당자가 B회사의 주문 번호(ID: 999)를 알아냄
- When: 대시보드 API 호출 시 URL을 억지로 `/api/v1/orders/999` 로 변조하여 GET 요청함
- Then: `OrderRepository` 수준에서 이미 `WHERE tenant_id = 'TENANT_A'` 가 강제 캐스팅되어 붙으므로, 내부 쿼리 결과값이 None 처리되며 결국 HTTP 404 Not Found 또는 403 Forbidden 이 떨어져 타사 데이터 유출을 원천에 막아낸다 (테스트 통과).

## ⚙️ Technical Constraints
- 이 테스트 케이스들은 추후 어떤 담당자가 미들웨어 로직(`DB-011`)을 부주의하게 훼손했을 때 가장 먼저 Red(실패)가 뜨며 치명적 배포를 막아야 하므로 CI 파이프라인에서 최우선으로 돌게끔 제어한다.

## 🏁 Definition of Done (DoD)
- [ ] 서로 다른 2개의 `tenant_id` 로 데이터를 섞어 구성한 Mock 환경 세팅 유무?
- [ ] 타겟 데이터(`get_by_id`) 접근 불가(Null 반환) Assertion 코드 작성 처리?
