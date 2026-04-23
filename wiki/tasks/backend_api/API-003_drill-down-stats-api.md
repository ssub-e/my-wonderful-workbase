---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] API-003: 드릴다운 뷰 전용 '일별/SKU별 핵심 요약' 집계 엔드포인트"
labels: 'feature, backend, api, priority:medium'
assignees: ''
type: task
tags: [task, backend_api]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [API-003] 특정 일자 정밀 데이터 조회를 위한 드릴다운(Drill-down) 전용 서버 API
- 목적: `VIEW-019` 의 요구사항에 맞춰, 대시보드 전체 기간 조회가 아닌 '특정 단일 일자' 에 집중된 SKU별 기여도 비중, 기상 변수가 해당 일자에 끼친 구체적 수치, 센터별 인원 역산 결과 등을 한 번의 요청으로 묶어서 반환한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- SRS 문서: [`e:\workspace\SRS-from-PRD\SRS-V1.0.md#6.1 API Endpoint List`](#)
- 데이터 모델: `FORECAST`, `FORECAST_FACTOR`, `ORDER_ITEM`

## ✅ Task Breakdown (실행 계획)
- [ ] `GET /api/v1/dashboard/drill-down?date=YYYY-MM-DD` 엔드포인트 신설
- [ ] 해당 일자의 `FORECAST` 데이터를 조회하여 전체 예측 수량 합산 로직 작성
- [ ] `FORECAST_FACTOR` 를 JOIN 하여 해당 일자의 Top 1 기여 변수(예: 강수량) 추출
- [ ] `ORDER_ITEM` 을 기반으로 해당 일자 주문 비중이 가장 높은 상위 5개 SKU 집계 SQL 작성
- [ ] 위 정보들을 하나로 결합한 `DrillDownResponseDTO` 정의 및 반환 (JSON)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 정상적인 일자 데이터 요청
- Given: DB에 2026-11-12 예측 데이터가 존재함
- When: `date=2026-11-12` 파라미터로 API를 호출함
- Then: `200 OK` 응답과 함께 `{ total_qty: 1540, top_skus: [...], primary_factor: "Rainfall" }` 형태의 압축된 데이터가 500ms 이내에 반환된다.

Scenario 2: 데이터 미생성 일자 요청 시 빈 객체 반환
- Given: 아직 수집/예측이 진행되지 않은 미래 날짜 (예: 2030-01-01)
- When: 해당 날짜로 드릴다운 요청을 보냄
- Then: `404 Not Found` 대신 `200 OK` 에 빈 배열이나 Null 필드를 포함한 객체를 반환하여 클라이언트 에러 바운더리(`VIEW-015`)가 부드럽게 처리할 수 있도록 돕는다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 복잡한 JOIN 쿼리 최적화 (p95 ≤ 300ms 달성). 필요 시 통합 통계 테이블(Materialized View) 사용 검토
- 보안: `AUTH-002` (멀티테넌트) 필터 강제 적용. 타사 날짜 데이터를 호출 시 무조건 빈 결과 전송
- 안정성: 에러율 ≤ 0.5% 유지

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 특정 날짜의 SKU 비중 합계가 100% (또는 전체 수량과 일치) 가 맞는지 데이터 검증 완료?
- [ ] Swagger API 문서에 해당 엔드포인트 명세 보강 완료?

## 🚧 Dependencies & Blockers
- Depends on: `DB-007`, `DB-003`
- Blocks: `VIEW-019` (프론트엔드 드릴다운 구현)
