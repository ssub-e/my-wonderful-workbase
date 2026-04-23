---
type: concept
tags: [adr, nfr, architecture-decision, non-functional-requirements]
created: 2026-04-22
updated: 2026-04-22
---
# ADR (아키텍처 결정 기록) 및 NFR (비기능 요구사항)

프로젝트 기술 결정과 성능/보안 관련 제약사항들입니다.

## 1. 주요 아키텍처 결정 (ADR)
* **ADR-01. MLaaS 배제 및 자체 통합망 구축 (Build)**: AWS Forecast 같은 관리형 서비스 대신 자체 시계열 엔진(In-house)을 구축합니다. 사유는 "SHAP 커스텀 변수 추출 등 윗선을 설득하기 위한 XAI(설명 가능한 AI) 해설 능력의 극대화"를 달성하기 위함입니다.
* **ADR-02. 복잡한 엑셀 엔진 대신 PDF 차용 (Buy)**: 내부망의 보수적인 결재 라인 통과를 위해, 웹뷰 UI를 Puppeteer 라이브러리로 렌더링/캡처하여 A4 용지 규격의 PDF 리포트로 출력합니다.

## 2. 비기능 요구사항 (NFR)
* **성능 (Performance)**: 
  * 리포트 계산 및 로딩: P95 기준 10초 이내
  * PDF 생성: P95 기준 20초 이내
* **신뢰성 (Reliability)**: 
  * 코어 서비스 SLA 가용성 99.5% 이상. 
  * ETL 파이프라인 실패 시 3회 자동 재시도 룰 적용. API Rate Limit 초과 시 즉각 Queue 모드로 전환.
* **보안 (Security)**: 논리적 멀티테넌시(Multi-tenancy) 기반의 엄격한 데이터 격리 및 JWT Role 분리 접근 제어.

## Related
- [기술 아키텍처 다이어그램 및 파이프라인](wiki/concepts/system_arch/tech-architecture.md)
- [원본 출처: PRD v1.1](wiki/sources/specification/source-prd-v1.1.md)
