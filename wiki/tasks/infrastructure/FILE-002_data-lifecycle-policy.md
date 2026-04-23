---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] FILE-002: S3 리포트 자산 보존을 위한 '데이터 생명주기(Lifecycle) 및 버저닝' 정책"
labels: 'feature, backend, infra, priority:low'
assignees: ''
type: task
tags: [task, infrastructure]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [FILE-002] 자동 자산 정리 및 문서 무결성 보호 정책
- 목적: REQ-NF-001 준수. 시스템에서 생성되는 방대한 PDF 리포트(`REPORT-001`)와 임시 파일들이 무한정 쌓여 스토리지 비용을 낭비하는 것을 막고, 실수로 삭제된 중요 문서를 복구할 수 있도록 S3 버킷 레벨의 생명주기 관리 및 버전 관리(Versioning) 기능을 활성화한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 스토리지: `FILE-001` (S3 Adapter)
- 관련 태스크: `INFRA-007` (Log Archiving)

## ✅ Task Breakdown (실행 계획)
- [ ] **S3 Versioning Enablement**: 
  - 모든 테넌트 자산 버킷에 대해 '버전 관리' 활성화 (실수 삭제 및 덮어쓰기 방어)
- [ ] **Lifecycle Policy (Report)**:
  - 생성된 지 1년이 지난 PDF 리포트는 자동으로 `Standard-IA` (저빈도 접근) 티어로 전환
  - 3년이 지난 문서는 자동으로 `S3 Glacier` 이관 또는 삭제 처리
- [ ] **Cleanup Worker (Temporary)**:
  - `/tmp/` 경로에 생성된 24시간 이상 된 중간 변환 파일들 자동 영구 삭제 규칙 적용
- [ ] **Integrity Check**: 파일 업로드 시 `Content-MD5` 체크섬을 사용하여 전송 중 데이터 오염 여부 검증
- [ ] **Inventory Report**: 매월 스토리지 용량 변동 및 점유율을 엑셀로 뽑아주는 모니터링 배치 추가

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 파일 삭제 후 버전 복구
- Given: 중요한 PDF 리포트가 S3에 저장됨
- When: 관리자 도구에서 실수로 해당 파일을 삭제(Delete) 함
- Then: 파일은 화면에서 사라지되, S3 콘솔의 '삭제 마커'를 제거하여 즉시 이전 버전으로 복구할 수 있어야 한다.

Scenario 2: 자동 비용 최적화 (Lifecycle)
- Given: 1년 1개월 전에 생성된 대용량 리포트 덤프
- When: S3 정책 점검 주기 도래
- Then: 해당 파일들은 자동으로 `Standard` 티어에서 비용이 저렴한 `Standard-IA` 티어로 물리적으로 이동되어야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 비용: 버전 관리는 스토리지 비용을 가중시키므로, 실시간 이미지가 아닌 '문서/설정' 파일에 대해서만 선별 적용
- 보안: Lifecycle 정책에 의해 삭제된 데이터는 복구가 불가능함을 명시하고 사전에 `SUPER_ADMIN`에게 고지
- 규정: 금융/의료 등 특정 산업군 고객사를 위해 5년 이상 '변경 불가(Object Lock)' 옵션 지원 가능 구조 설계

## 💻 Implementation Snippet (S3 Lifecycle Rule JSON Idea)
```json
{
  "Rules": [
    {
      "ID": "MoveReportsToIA",
      "Status": "Enabled",
      "Filter": { "Prefix": "reports/" },
      "Transitions": [
        {
          "Days": 365,
          "StorageClass": "STANDARD_IA"
        }
      ]
    },
    {
      "ID": "CleanTmpFiles",
      "Status": "Enabled",
      "Filter": { "Prefix": "tmp/" },
      "Expiration": { "Days": 7 }
    }
  ]
}
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] AWS CLI/SDK를 통해 Lifecycle 정책이 버킷에 올바르게 주입되었는가?
- [ ] 불필요한 버전이 과도하게 쌓이지 않도록 '비최신 버전 보존 기간' 설정이 완료되었는가?

## 🚧 Dependencies & Blockers
- Depends on: `FILE-001`
- Blocks: 없음
