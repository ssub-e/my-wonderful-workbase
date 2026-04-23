---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[SaaS-Admin] SUPER-004: 과금 요금제(Plan) 기반 API 호출 Rate Limit 미들웨어"
labels: 'backend, security, saas-admin, phase:8, priority:low'
assignees: ''
type: task
tags: [task, admin]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [SUPER-004] 무료(Free) 플랜 유저의 과도한 API 콜 및 AI 자원 탈취 방어
- 목적: SaaS의 안정성과 요금 차등을 보장하기 위해, B2B 고객사의 요금제 등급(`SUPER-001` 에서 정의한 `FREE`, `PRO`)을 인식하여 무거운 연산(AI XAI 추론 등) API 로의 진입을 막거나, 1분당 호출 제한을 건다.

## 🔗 References (Spec & Context)
- 타겟 플랜 변수: `SUPER-001` (`BillingPlanEnum.FREE`)
- 기술 모듈: FastAPI 미들웨어, 혹은 Redis(`INFRA-002`) 토큰 버킷 활용

## ✅ Task Breakdown (실행 계획)

### 1. 라우터별 요금제 등급 차단 계층 (Dependency)
- [ ] `app/api/deps.py` 확장:
  ```python
  from fastapi import Depends, HTTPException

  def get_current_active_pro_tenant(current_user = Depends(get_current_user)):
      # 캐싱된 테넌트 메타데이터에서 요금제 확인 가정
      tenant_plan = fetch_tenant_plan_from_memory(current_user.tenant_id)
      
      if tenant_plan == "FREE":
          raise HTTPException(
             status_code=402, # Payment Required
             detail="해당 기능은 PRO 혹은 ENTERPRISE 요금제 전용입니다. 플랜을 업그레이드 해주세요."
          )
      return current_user
  ```
- 이후 XAI 분석 라우터(`DASH-003`) 쪽에 `dependencies=[Depends(get_current_active_pro_tenant)]` 라고 붙여주기만 하면 우주 방어가 완성된다.

### 2. (선택) Rate-Limiter (호출량 폭주 제한)
- 프리미어 요금제가 아닌 곳에서 무차별 클릭으로 서버 DB를 폭파하거나 클라우드 대역폭 비용을 높이는 것을 막으려면 Redis를 이용한 Rate Limiting 도입을 설계서 단에 권고조치 한다 (`SlowAPI` 라이브러리 연동). 

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 무료 플랜 고객의 PRO 기능 해킹 탈취 방지**
- Given: `FREE` 플랜에 속한 고객사가 회원가입시 받은 Access Token 으로 로그인함.
- When: 프론트엔드가 아닌 Postman을 통해 고강도 AI 연산 엔드포인트 `/api/v1/xai/predict` 로 직접 악의적 파싱을 시도함.
- Then: 이 기능을 지키고 있는 `get_current_active_pro_tenant` 문지기 로직에 의해 402 에러코드로 퇴짜 맞음으로써, 기업의 AI 서버 연산 자원을 비용 지불 없이 무단 포식하는 행위를 차단한다.

## ⚙️ Technical Constraints
- 매 API 통신마다 `Tenant` 테이블에서 `Plan` 속성을 찾기 위해 SQL 조회를 날리면 전체 시스템 병목(`N+1` 유사 쿼리 과부하)이 일어날 수 있다. 즉 JWT 페이로드를 구울 때 아예 `plan` 을 박아 넣거나 Redis에 `TENANT_ID_PLAN` 형태로 빠르게 메모리 캐싱(`INFRA-002` 연계)되어 있어야 한다.

## 🏁 Definition of Done (DoD)
- [ ] 요금제 속성(ex: "FREE", "PRO") 에 따라 402 에러를 반환하는 디펜던시 룰 구성?
- [ ] 캐싱 지향적/혹은 토큰 속성 매칭 형태의 성능 저하 없는 검사기 권고 반영 여부?
