---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-017: 데이터 테이블 엑셀(CSV) 다운로드 뷰 액션"
labels: 'feature, frontend, priority:low'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-017] Data Grid 내용 엑셀(CSV) 클라이언트 사이드 파싱 및 다운로드
- 목적: B2B 사용자들이 "이 데이터 엑셀로 뽑아서 볼 수 없나요?" 라고 묻는 가장 클래식한 요구사항을 충족시키기 위해, 브라우저 단에서 렌더링된 JSON 배열을 CSV 포맷으로 변환해 로컬 다운로드를 발생시킨다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 대상 컴포넌트: `VIEW-008` (Workforce 역산 표), `VIEW-013` (Audit Log 표)
- 서드파티 라이브러리 검토: `xlsx` 또는 순수 JS Blob 파서

## ✅ Task Breakdown (실행 계획)
- [ ] 재사용성 높은 Client-side CSV 파싱 유틸 함수 (`src/utils/exportCsv.ts`) 구현
- [ ] 한글 깨짐 방지를 위한 UTF-8 (BOM 바이트 `\uFEFF` 삽입) 인코딩 로직 적용
- [ ] 브라우저 `URL.createObjectURL(blob)` 함수를 활용한 가상 `<a>` 태그 클릭 트리거 생성
- [ ] Data Grid 표 상단 컨트롤 패널 영역에 `Download Excel/CSV` 버튼 (Icon 포함) 배치
- [ ] 권한이 억제된 Guest/하위 권한 유저의 경우 버튼 비활성화 로직(RBAC 연동)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 검색된 현재 표의 데이터를 CSV로 추출
- Given: 대시보드 표에 "어제 주문 100건" 등의 데이터가 10줄 렌더링 되어 있음
- When: 우측 상단의 'CSV 다운로드' 버튼을 클릭함
- Then: 백엔드 서버를 거치지 않고, 화면에 보이는 JSON 데이터 그대로 변환된 `report_2026-11-12.csv` 파일이 즉각적으로 디바이스 로컬 하드에 저장된다. 파일을 엑셀로 열었을 때 한글이 깨지지 않아야 한다.

Scenario 2: 타 권한 유저의 다운로드 시도 방어
- Given: `VIEWER` (단순 열람 전용 알바생 계정) 로 로그인
- When: 대시보드에 접근함
- Then: CSV 다운로드 버튼 자체가 랜더링 되지 않거나(Disabled), '다운로드 권한이 없습니다' 라는 툴팁이 표출되며 데이터 자산 유출을 원천 방어한다.

## ⚙️ Technical & Non-Functional Constraints
- 성능: 응답시간 p95 ≤ 300ms 달성 (수만 줄의 데이터일 경우 클라이언트 메인스레드가 뻗지 않도록 Web Worker 권장되나 MVP는 100줄 이내로 끊으므로 미적용)
- 안정성: 에러율 ≤ 0.5% 유지
- 보안: 다운로드 자체는 클라이언트에서 일어나지만, 민감 정보를 다루는 만큼 권한(Token Role) 체계를 엄격히 마운팅할 것.

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 단위 테스트(Unit Test) 및 통합 테스트(Integration Test)가 추가되었고 통과하는가?
- [ ] SonarQube / Linter 등의 정적 분석 도구 경고가 없는가?
- [ ] API 명세서(Swagger 등)가 최신화되었는가? (이 경우엔 Frontend Util 검수)

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-008`, `VIEW-013` (대상 Data Grid 확보 필요)
- Blocks: 없음
