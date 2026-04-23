---
type: concept
tags: [architecture, system-design, airflow, ml, fastapi]
created: 2026-04-22
updated: 2026-04-22
---
# 기술 아키텍처 다이어그램 및 파이프라인

B2B 수요예측 AI SaaS 시스템의 구성 및 데이터 흐름 파이프라인입니다.

## 1. External Integration (데이터 파이프라인)
* **목표**: 외부 데이터의 지속적이고 안정적인 수집.
* **구성요소**: 기상청/포털 API 및 이커머스(Cafe24) API
* **오케스트레이션**: Apache Airflow를 통해 ETL 스케줄링. PostgreSQL 레이크에 적재.

## 2. Core Engine (ML 및 Backend)
* **데이터 흐름**: 
  1. Airflow 수집 데이터 -> PostgreSQL
  2. PostgreSQL -> **TimeSeries Model (N-BEATS 등 앙상블)**
  3. ML Model -> **SHAP Analyzer (XAI 해설 엔진)** 및 **인력 역산 최적화 로직**
* **API 제공**: FastAPI 백엔드 서버를 통해 추론 결과물과 리포트 제공.

## 3. Client Delivery (출력/제공 계층)
* **React Dashboard**: 사용자 인터페이스 (대시보드)
* **Puppeteer PDF Gen**: 결재용 리포트 파일 생성 (엑셀 대체용, 보수적 환경 대응)
* **카카오 알림톡 발송 모듈**: 인력소/담당자 대상 출고 알바 인원 자동 산정/통보

## Related
- [ADR 및 NFR 제약사항](wiki/concepts/system_arch/adr-nfr.md)
- [데이터베이스 ERD](wiki/entities/database/erd-core-entities.md)
- [원본 출처: PRD v1.1](wiki/sources/specification/source-prd-v1.1.md)
