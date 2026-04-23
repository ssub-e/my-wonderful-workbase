---
type: source
tags: [source-summary, srs, specification, requirements]
created: 2026-04-22
updated: 2026-04-22
---
# 요약본: SRS-V1.0

PRD v1.1을 기반으로 작성된 소프트웨어 요구사항 명세서(SRS-V1.0)의 핵심 요약본입니다. 이 문서는 구체적인 기술 스택(C-TEC) 제약과 API 명세, 3대 핵심 기능(XAI 리포트, API 연동, 인원 역산)에 대한 비기능적 요구사항(NFR) 및 테스트 케이스를 포괄합니다.

## 포함된 원본 소스
- `SRS-V1.0.md` (ISO/IEC/IEEE 29148:2018 준수)

## 핵심 시사점 (Key Takeaways)
- **개발 스택의 단일화 (MVP)**: 복잡성을 줄이기 위해 MVP 단계에서는 JavaScript 프론트엔드를 배제하고, Python 기반의 **FastAPI + Streamlit + LightGBM + PostgreSQL** 스택으로 통일합니다.
- **철저한 Fallback 전략**: 기상청 API 장애, 화주사 쇼핑몰 인증 만료, 카카오 알림톡 전송 실패 시 각각 캐시 반영, 생존 화주 데이터만 계산, SMS 1분 내 전송 등의 우회(Fallback) 시나리오가 의무화되어 있습니다.
- **멀티테넌트 보안 격리**: 여러 B2B 화주사의 데이터가 한 DB에 모이므로, `tenant_id` 기반 Row-Level Isolation이 강제되며 데이터 교차 노출은 시스템상 치명적 오류로 간주됩니다.

## 파생된 Wiki 노드
- [API 및 연동 스펙 명세서](wiki/concepts/system_arch/api-specs.md)
- [C-TEC 기술 스택 및 제약사항](wiki/concepts/system_arch/tech-stack-srs.md)
- [원본 출처: PRD v1.1 요약](wiki/sources/specification/source-prd-v1.1.md)

## Related
- [원본 폴더 위치: raw/assets/](raw/assets/)
