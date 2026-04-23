---
type: concept
tags: [api, system-architecture, interface]
created: 2026-04-22
updated: 2026-04-22
---
# API 및 연동 스펙 명세서

SRS-V1.0에 기반하여 정리된 내부 API(Internal)와 외부 API(External) 연결망 아키텍처입니다.

## 1. Internal API (내부 백엔드)
FastAPI를 통해 구축되는 핵심 마이크로서비스 지향 엔드포인트입니다.
- **예측 엔진 (`/api/v1/forecast`)**: LightGBM 모델 추론. 추론 시간 p95 30초 이내 보장.
- **XAI 해설 (`/api/v1/xai/explain`)**: SHAP 기여도 Top 3 추출 및 LLM 자연어 해설 렌더링.
- **PDF 렌더링 (`/api/v1/report/generate`)**: WeasyPrint를 활용한 결재용 보고서 바이너리 생성. 생성 시간 p95 20초 이내 보장.
- **쇼핑몰 OAuth (`/api/v1/integration/shop`)**: 1-Click 인증 URL 발급 및 토큰 교환 콜백 처리.
- **인원 역산 (`/api/v1/workforce/calculate`)**: 예측 출고량 기반 센터별 권장 투입 인력(명) 도출.
- **알림톡 전송 (`/api/v1/notification/send`)**: 센터장 및 인력소로 매일 16시 정각 전송. 실패 시 1분 내 SMS 단문 폴백.

## 2. External API (외부 Inbound/Outbound)
시스템이 가져오거나(Inbound) 내보내는(Outbound) 주요 서드파티 서비스입니다.
- **기상청 단기예보 API (Inbound)**: 일 1회 메인 배치 덤프. Rate Limit 일 1,000회 제한 주의. 장애 시 24시간 이내 캐시로 Fallback.
- **네이버 DataLab (Inbound)**: 트렌드 검색량 지수. 장애 시 30일 이력 대체.
- **카페24 / 스마트스토어 API (Inbound)**: 분당 호출 Rate Limit 대응을 위해 캐시 및 큐잉 필수. 화주 인증 토큰 만료 시 부분 연동 끊김(에러 무시)으로 대응.
- **카카오 알림톡 (Outbound)**: 템플릿 기반 발송. 장애 시 즉각 SMS 알림망으로 전환.

## Related
- [기술 아키텍처 다이어그램 및 파이프라인](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/tech-architecture.md)
- [C-TEC 기술 스택 및 제약사항](file:///e:/workspace/my-wonderful-workbase/wiki/concepts/system_arch/tech-stack-srs.md)
- [원본 출처: SRS-V1.0 요약](file:///e:/workspace/my-wonderful-workbase/wiki/sources/specification/source-srs-v1.0.md)
