---
type: entity
tags: [database, erd, schema, prd]
created: 2026-04-22
updated: 2026-04-22
---
# 데이터베이스 코어 엔티티 (ERD)

수요예측 AI SaaS의 논리적 데이터베이스 구조를 정의하는 핵심 엔티티들입니다. (PRD v1.1 기준)

## 핵심 테이블 관계 (ERD)
* **TENANT (고객사)**
  * `SHOP` (1:N) - 운영하는 쇼핑몰/화주사
  * `FORECAST` (1:N) - 생성된 발주 예측치
  * `WORKFORCE_PLAN` (1:N) - 출하 인력 계획
* **SHOP (플랫폼)**
  * `ORDER` (1:N) - 누적된 주문 및 매출
  * `PRODUCT` (1:N) - 취급 품목
* **FORECAST (수요 예측 결과)**
  * `FORECAST_FACTOR` (1:N) - 예측 결과에 영향을 미친 외부 변인들(XAI SHAP 지분)

## 주요 필드
* **TENANT**: `id` (PK, uuid), `name`, `plan_tier`
* **SHOP**: `platform` (예: cafe24, smartstore), `oauth_token`
* **FORECAST**: `predicted_qty`, `confidence_level` (AC 1.3 트리거 기준선)
* **FORECAST_FACTOR**: `factor_type` (날씨, 트렌드 등), `shap_value` (XAI 인과 지분)

## Related
- [기술 아키텍처 및 시스템 파이프라인](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/tech-architecture.md)
- [원본 출처: PRD v1.1](file:///e:/workspace/my-wonderful-workbase/wiki/sources/specification/source-prd-v1.1.md)
