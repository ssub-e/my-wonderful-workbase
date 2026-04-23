---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] TEST-005: 서비스 품질 보증을 위한 '핵심 비즈니스 시나리오 E2E 자동화 테스트'"
labels: 'feature, backend, frontend, qa, priority:medium'
assignees: ''
type: task
tags: [task, testing]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [TEST-005] Playwright 기반 임계 경로(Critical Path) E2E 테스트 자동화
- 목적: 배포 전후에 실제 브라우저 환경에서 서비스의 주요 기능(로그인 → 데이터 수집 → 예측 조회 → 리포트 생성)이 정상적으로 동작하는지 기계적으로 시뮬레이션하여 휴먼 에러를 방지한다. 이는 복잡한 싱글 페이지 애플리케이션(SPA)의 사이드 이펙트를 잡아내는 가장 강력한 보루다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 라이브러리: `Playwright`, `Vitest`
- 관련 태스크: `TEST-003~004` (UI Unit Test), `DEPLOY-004` (CD 연동)

## ✅ Task Breakdown (실행 계획)
- [ ] **Playwright Environment Setup**:
  - 크롬, 파이어폭스, 웹킷 브라우저 지원 및 해상도(Desktop/Mobile)별 설정
- [ ] **Auth Flow Test**: 로그인 페이지 진입부터 토큰 발급 및 대시보드 리다이렉션 검증
- [ ] **Data Pipeline Trigger Test**: 
  - 관리자 설정 탭에서 '수동 수집' 버튼 클릭 시 백그라운드 작업이 시작되고 알림이 뜨는지 확인
- [ ] **Dashboard Integrity Test**: 차트가 로딩 스켈레톤을 거쳐 비어있지 않은 상태로 렌더링되는지 Selector 기반 검증
- [ ] **Report Creation Flow Test**: PDF 생성 버튼 클릭 시 팝업이 뜨고 파일이 다운로드되는지 전체 경로 추적
- [ ] **CI Integration**: GitHub Actions 빌드 성공 직후 스테이징 환경에서의 테스트 자동 실행 및 결과 리포트 업로드

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 로그인부터 대시보드 도달까지 성공 시나리오
- Given: 테스트용 계정 정보
- When: 웹 페이지 접속 후 이메일/PW 입력 및 로그인 클릭
- Then: URL이 `/dashboard` 로 변경되며, KPI 수치들이 화면에 정확히 표시되어야 한다.

Scenario 2: 모바일 뷰에서의 햄버거 메뉴 동작
- Given: 모바일 뷰포트(iPhone 13) 설정
- When: 햄버거 버튼 클릭 후 메뉴 항목 선택
- Then: 메뉴가 사이드에서 부드럽게 나타나고, 페이지 이동이 정상적으로 이루어져야 한다.

## ⚙️ Technical & Non-Functional Constraints
- 안정성: 네트워크 지연 등으로 인한 테스트 실패를 막기 위해 `waitForSelector` 와 같은 명시적 대기 전략 사용
- 성능: 테스트 수행 시간을 단축하기 위해 병렬 시나리오 실행(Parallel Execution) 적용
- 데이터 격리: 테스트 수행 때마다 깨끗한 상태의 데이터베이스 픽스처(`SEED_DATA`)를 로드하여 불확실성 제거

## 💻 Implementation Snippet (Playwright Test Code Idea)
```javascript
import { test, expect } from '@playwright/test';

test('Full flow: Login to Report', async ({ page }) => {
  await page.goto('/login');
  await page.fill('input[name="email"]', 'test@example.com');
  await page.fill('input[name="password"]', 'password123');
  await page.click('button[type="submit"]');

  await expect(page).toHaveURL('/dashboard');
  await page.click('text=Generate Report');
  
  const downloadPromise = page.waitForEvent('download');
  await page.click('button:has-text("Confirm")');
  const download = await downloadPromise;
  expect(download.suggestedFilename()).toContain('.pdf');
});
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] CI 파이프라인에서 E2E 테스트 실패 시 배포가 자동으로 중단되는가?
- [ ] 테스트 실행 중 캡처된 비디오 또는 스크린샷이 아티팩트로 저장되는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-004`, `DEPLOY-001`
- Blocks: 없음
