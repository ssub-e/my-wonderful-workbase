---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Frontend] VIEW-014: 비동기 리포트 생성 대기창(Polling) 및 URL 오픈 컴포넌트"
labels: 'frontend, view, report, phase:5, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-014] 대시보드 내 PDF 리포트 다운로드 진행 상태 UI 
- 목적: REQ-FUNC-007의 파일 엑스포트 체계를 완성한다. 사용자가 PDF 버튼을 누르고 `REPORT-002` 가 준 202 Accepted 응답(`job_id`)을 캐치한 후, 약 2~4초 간격으로 진행 상황을 폴링(Polling)하다가 완성되면 S3가 서명해 발급한 보안 링크를 새 창으로 자동으로 열어버린다.

## 🔗 References (Spec & Context)
- 연결 API: `REPORT-002` (POST `/report/generate` 및 GET `/report/{job_id}/status`)
- 연관 UI: `대시보드 상단 우측 다운로드 버튼`

## ✅ Task Breakdown (실행 계획)

### 1. Polling 컨트롤 커스텀 훅
- [ ] `src/hooks/useReportGenerator.ts`:
  ```typescript
  import { useState } from 'react';
  import { apiClient } from '@/api/client';

  export const useReportGenerator = () => {
    const [isGenerating, setIsGenerating] = useState(false);

    const generateAndDownload = async (targetDate: string) => {
      setIsGenerating(true);
      try {
        // 1. 비동기 Task 접수증(job_id) 발급
        const { data: initData } = await apiClient.post(`/report/generate?target_date=${targetDate}`);
        const jobId = initData.job_id;

        // 2. 상태 폴링 타이머 루프 (while)
        while (true) {
          await new Promise(resolve => setTimeout(resolve, 2000)); // 2초 대기
          
          const { data: statusData } = await apiClient.get(`/report/${jobId}/status`);
          
          if (statusData.status === "generated") {
            // S3 Signed URL 을 즉각 새 창으로 오픈하여 파싱된 PDF 렌더
            window.open(statusData.download_url, '_blank');
            break;
          } else if (statusData.status === "failed") {
            alert('PDF 생성 도중 시스템 예외가 발생했습니다.');
            break;
          }
          // processing 상태면 루프 계속 순회
        }
      } catch (err) {
        alert('요청 접수에 실패했습니다.');
      } finally {
        setIsGenerating(false);
      }
    };

    return { generateAndDownload, isGenerating };
  };
  ```

### 2. 트리거 버튼 컴포넌트
- [ ] `src/components/dashboard/DownloadReportButton.tsx`:
  - `<Button onClick={() => generateAndDownload(today)} disabled={isGenerating}>` 와 같이 바인딩하고 `isGenerating` 이 True일때 버튼 레이블을 `PDF 구워내는 중... ⏳` 과 같은 스피너로 바꾼다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 서버 Polling 루프 정상 해제**
- Given: `generateAndDownload` 함수가 트리거되어 2초마다 핑(ping)이 나가는 중.
- When: 백엔드가 10초 뒤 `{"status": "generated"}` 및 다운로드 링크를 리턴함.
- Then: `while` 문 밖으로 `break` 되어 폴링 루프가 영구히 해제됨과 즉시, 브라우저의 기본 설정인 팝업(`window.open`) 차단 조건에 걸리지 않도록 명확한 동선 설계가 이루어져 파일이 열리게 된다.

## ⚙️ Technical Constraints
- SPA 상에서 무한 루프(while true)는 위험할 수 있으므로, 최대 폴링 횟수(예: 30회 = 1분)가 넘어가면 억지로 `Timeout 에러`를 던지고 `break` 해버리는 안전장치 카운터(Counter)를 반드시 포함한다.

## 🏁 Definition of Done (DoD)
- [ ] `while` 과 `setTimeout` 을 이용한 Non-UI 블로킹 백그라운드 폴링 함수가 작성되었는가?
- [ ] 최대 대기 시간(Timeout) 제어 변수가 구현되었나?
- [ ] `window.open()`을 통한 안전한 S3 Presigned URL 오픈 로직 반영부?
