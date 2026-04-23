---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] MONITOR-004: 사용자 경험 개선을 위한 '인앱 피드백 및 버그 제보' 위젯"
labels: 'feature, frontend, monitoring, priority:low'
assignees: ''
type: task
tags: [task, monitoring]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [MONITOR-004] 사용자 보이스 수집 및 이슈 트래킹 위젯
- 목적: REQ-NF-001 보장. 사용자가 서비스 이용 중 불편함이나 버그를 발견했을 때, 별도의 메일 전송 없이 화면 우측 하단 위젯을 통해 즉시 스크린샷과 함께 제보할 수 있는 채널을 마련한다. 이는 CS 비용을 절감하고 제품 퀄리티를 능동적으로 개선하는 핵심 창구가 된다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 인프라: `MONITOR-001` (Slack Alert)
- 스토리지: `FILE-001` (스크린샷 저장)

## ✅ Task Breakdown (실행 계획)
- [ ] **Feedback Trigger Overlay**: 화면 우측 하단에 떠 있는 플로팅 버튼(Floating Button) 구현
- [ ] **Modal Form**: 
  - 피드백 분류(버그/제안/칭찬), 내용 입력칸
  - 현재 화면 자동 캡처(html2canvas 활용) 및 첨부 기능
- [ ] **Feedback API**: 
  - `POST /api/v1/support/feedback`
  - 유저 세션 정보(ID, Tenant, 브라우저 버전, 현재 URL)와 함께 DB 저장
- [ ] **Immediate Notification**: 제보 접수 시 지정된 운영팀 슬랙 채널(`MONITOR-001`)로 상세 정보 및 스크린샷 링크 즉시 전송
- [ ] **Admin Dashboard View**: 슈퍼 어드민이 제보된 피드백 목록을 조회하고 '처리 중/완료' 상태를 관리하는 뷰

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 버그 제보 수행
- Given: 대시보드 오류를 발견한 유저
- When: 피드백 위젯을 통해 "그래프가 안 나와요"라고 작성하고 스크린샷과 함께 제출함
- Then: 서버에 성공적으로 저장되며, 운영팀 슬랙에 "테넌트 A로부터 새로운 버그 제보가 도착했습니다" 라는 알림과 함께 해당 유저의 정확한 뷰포트 정보가 노출되어야 한다.

Scenario 2: 자동 메타데이터 수집 확인
- Given: 피드백 제출 완료
- When: 관리자가 DB를 확인함
- Then: 유저가 직접 입력하지 않아도 `User-Agent`, `OS`, `Current_URL` 정보가 함께 저장되어 기술 지원이 가능해야 한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: 서비스 본연의 기능(차트 확인 등)을 가리지 않도록 위젯 위치를 유동적으로 조정하거나 숨길 수 있게 함
- 개인정보: 피드백 본문에 포함될 수 있는 PII 데이터 주의 및 스크린샷 내 개인 정보 마스킹 안내 문구 노출
- 가용성: 피드백 시스템 자체의 장애가 메인 서비스 로딩을 방해하지 않도록 별도의 비동기 스크립트로 로드

## 💻 Implementation Snippet (Screenshot Capture Idea)
```javascript
import html2canvas from 'html2canvas';

const captureAndSubmit = async (feedbackData) => {
    const canvas = await html2canvas(document.body);
    const screenshot = canvas.toDataURL("image/png");
    
    // API Call
    await api.post("/support/feedback", {
        ...feedbackData,
        screenshot: screenshot, // Large blob
        meta: {
            url: window.location.href,
            ua: navigator.userAgent
        }
    });
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] 슬랙 알림에 스크린샷 이미지가 정상적으로 임베디드되어 보이는가?
- [ ] 제출 후 유저에게 "제보해 주셔서 감사합니다" 라는 피드백 토스트가 노출되는가?

## 🚧 Dependencies & Blockers
- Depends on: `MONITOR-001`, `FILE-001`
- Blocks: 없음
