---
type: concept
tags: [competitor-analysis, market-research, business-strategy]
created: 2026-04-22
updated: 2026-04-22
---
# 경쟁사 포지셔닝 및 GTM 전략

솔루션 구축과 직결되는 핵심 3대 연관 시장의 경쟁 구도와 우리의 생존 전략(ADR 연계)입니다.

## 1. MLOps 인프라 시장
- **주요 벤더**: AWS SageMaker, Databricks, 베슬에이아이(VESSL AI), 마키나락스
- **우리의 전략 (Buy vs Build)**: 예측 정확도가 핵심 경쟁력이므로 파이프라인(K8s 등) 운영 인프라 구축에 개발 리소스를 낭비(Build)하지 않음. AWS나 베슬에이아이 같은 기성 인프라를 도입(Buy)하여 자본 연소(Burn Rate)를 막아야 함.

## 2. SCM / ERP 소프트웨어 시장
- **주요 벤더**: SAP, Blue Yonder, 엠로(emro), 더존비즈온, 영림원
- **우리의 전략 (Add-on 기생)**: 대기업 실무자는 새로운 툴이나 대시보드 화면을 띄우는 것을 극도로 꺼림. 독립적인 대시보드 스탠드얼론(Standalone) 판매를 지양하고, 기존 ERP/SCM 시스템 뒤에서 돌아가는 **API 플러그인(Add-on) 형태** 또는 친숙한 형태(PDF/알림톡)로 전략을 수정해야 함. 포괄적인 협력업체 관리는 버리고 오직 '수치/정확도 개선' 스펙 하나에 100% 집중.

## 3. 대안 데이터 (Alternative Data) 시장
- **주요 벤더**: YipitData, FactSet, 딥서치, ZoomInfo
- **우리의 전략 (Curation)**: 초기부터 직접 웹 크롤링 파이프라인을 구축하지 않고, 이미 정제된 대안 데이터(기상, 경제 트렌드 등)를 제공하는 핵심 API 벤더와 제휴하여 프록시 형태로 연동함. 단순 차트 뷰가 아닌 "내일 줄일 수 있는 인건비/절감 차액"과 같은 **액션 가능한 시그널(Actionable Signal)**을 최종 결과물로 제공해야 함.

## Related
- [5 Forces 시장 매력도 분석](wiki/concepts/business_strategy/five-forces-market.md)
- [원본 출처: 경쟁분석 종합 (02)](wiki/sources/business/source-market-competitor.md)
