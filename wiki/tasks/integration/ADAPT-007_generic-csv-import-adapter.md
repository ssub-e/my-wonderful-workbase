---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] ADAPT-007: 비표준 시스템 대응을 위한 '범용 ERP/WMS CSV 데이터 임포트' 보일러플레이트"
labels: 'feature, backend, adapter, priority:low'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-007] 수동 데이터 연동을 위한 유연한 CSV 어댑터
- 목적: 카페24 같은 자동 API 연동이 불가능한 자체 구축(Legacy) ERP를 사용하는 고객사나 엑셀 위주로 관리하는 영세 창고를 위해, 컬럼 맵핑을 통해 어떠한 형태의 CSV 파일이라도 시스템 표준 포맷으로 변환하여 밀어넣어 줄 수 있는 제너릭 수집기(Boilerplate)를 제공한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 데이터 정제: `ETL-001` (ETL 파이프라인)
- 관련 태스크: `VIEW-011` (수동 업로드 UI)

## ✅ Task Breakdown (실행 계획)
- [ ] **Configurable Schema Mapper**:
  - 고객사마다 다른 CSV 컬럼명(예: "상품명" vs "Product_Name")을 우리 시스템의 `product_name` 필드로 연결하는 JSON 설정 구조 구축
- [ ] **Fast Stream Parser**: 
  - 대용량 파일 업로드 시 메모리 초과 방지를 위해 `Lazy Parsing` (Python `csv` 모듈 또는 `Pandas chunksize`) 적용
- [ ] **Validation & Cleaning Engine**: 유효하지 않은 날짜 포맷, 특수 문자, 누락된 필수값을 전처리하고 오류 행만 따로 리포트하는 기능
- [ ] **Batch Insert Logic**: 정제된 데이터를 `INFRA-004`(Bulk Loader)를 통해 DB에 광속 적재
- [ ] **Template Generator**: 데이터 입력 편의를 위해 '시스템 권장 엑셀 양식'을 다운로드 받는 엔드포인트

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 비표준 CSV 업로드 및 맵핑
- Given: 컬럼명이 "주문번호", "판매량", "날짜" 로 된 외부 ERP 엑셀 파일
- When: 맵핑 가이드(Mapping JSON)를 적용하여 업로드 수행
- Then: 시스템은 해당 한글명을 인식하여 정해진 필드에 정확히 데이터를 치환하여 적재해야 한다.

Scenario 2: 데이터 오염 시 부분 성공 처리
- Given: 1,000행 중 10행의 날짜 형식이 깨진 파일
- When: 임포트 실행
- Then: 나머지 990행은 성공적으로 DB에 들어가고, 실패한 10행의 사유와 원본 데이터를 담은 '오류 결과 파일'을 유저에게 반환해야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 10MB 이상의 파일은 백엔드에서 `BackgroundTasks` 로 처리하고 완료 시 알림톡 전송
- 인코딩: 한글 엑셀 특유의 `CP949` 와 `UTF-8` 인코딩을 자동 감지하여 깨짐 방지
- 보안: 실행 파일 주입 등 악의적인 파일 업로드 방지를 위한 엄격한 MIME Type 체크

## 💻 Implementation Snippet (Generic CSV Mapping Idea)
```python
def process_custom_csv(file_stream, mapping_config):
    """
    mapping_config: {"order_no": "주문번호", "qty": "판매수량"}
    """
    for chunk in pd.read_csv(file_stream, chunksize=1000, encoding='utf-8-sig'):
        # 컬럼명 치환
        renamed_chunk = chunk.rename(columns={v: k for k, v in mapping_config.items()})
        # 우리 시스템 필수 필드만 추출
        cleaned_data = renamed_chunk[list(mapping_config.keys())]
        # DB 적재 로직 호출
        bulk_save(cleaned_data)
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 컬럼 맵핑 정보가 테넌트별로 저장되어 재사용 가능한가?
- [ ] 수천 건의 데이터 실적 차트(`VIEW-006`)에 즉시 반영되는가?

## 🚧 Dependencies & Blockers
- Depends on: `INFRA-004`
- Blocks: 없음
