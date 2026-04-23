---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[API] REPORT-002: PDF 비동기 생성 요청 및 S3 다운로드 URL 발급 API"
labels: 'api, report, security, phase:3, priority:low'
assignees: ''
type: task
tags: [task, backend_core]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [REPORT-002] 결재 문서 생성 백그라운드 위임 및 다운로드 서명(Signed URL) API
- 목적: 클라이언트(프론트)가 "PDF 다운로드" 버튼을 눌렀을 때, 무거운 PDF 생성(`REPORT-001`) 객체를 비동기로 위임하고 DB 상태를 활용해 폴링(Polling) 대기 로직을 짤 수 있도록 지원. 완성 후엔 직접 파일 스트림을 꽂지 않고 S3나 Supabase Storage의 `Presigned URL`(보안 만료 링크)를 내어주어 서버 부하를 줄인다.

## 🔗 References (Spec & Context)
- SRS 아키텍처: PDF 파일 생성 및 외부 스토리지 적재 파이프라인
- 보안 제약 기준: REQ-NF-018 (서명된 URL 통한 접근 통제 - 권한자만 열람 가능)

## ✅ Task Breakdown (실행 계획)

### 1. 비동기 생성 요청 엔드포인트 세팅
- [ ] `app/api/endpoints/report.py`
  ```python
  from fastapi import APIRouter, BackgroundTasks, Depends
  import uuid

  @router.post("/generate", status_code=202)
  async def trigger_pdf_generation(
      background_tasks: BackgroundTasks,
      target_date: str,
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      """ 
      1. DB-008 `reports` 테이블에 status='processing' 비어있는 행 기록
      2. BackgroundTasks 에 PDF 렌더러 + S3 업로더를 묶은 태스크 위임
      3. 즉각적으로 row id(결과 참조용)만 클라이언트에 202 응답 리턴
      """
      # initiate_report() from report_repository 모듈
      report_row = await initiate_report(session, tenant_id, None, "temp")
      
      background_tasks.add_task(
          async_generate_and_upload_pdf, 
          report_row.id, tenant_id, target_date
      )
      return {"message": "Processing started", "job_id": str(report_row.id)}
  ```

### 2. URL 발급 및 폴링 체크용 엔드포인트
- [ ] 상태 체크 API 구현:
  ```python
  @router.get("/{job_id}/status")
  async def check_report_status(
      job_id: uuid.UUID,
      tenant_id: str = Depends(get_current_tenant),
      session: AsyncSession = Depends(get_session)
  ):
      """ 프론트에서 2초에 한번씩 찔러보는 체크용 API. 완료시 링크 발사 """
      r = await get_report(session, tenant_id, job_id)  # 교차노출 방어 내장 함수
      
      if r.status == "processing":
          return {"status": "processing"}
          
      elif r.status == "generated":
          # Boto3 (AWS S3) 또는 Supabase Python SDK를 활용하여 5분 한정 Presigned URL 발급
          # url = s3_client.generate_presigned_url(ClientMethod='get_object', ...)
          presigned_url = "https://s3.aws.com/your-bucket/secure-url?sign=xxxx"
          
          return {"status": "generated", "download_url": presigned_url}
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Non-Blocking 성능 보장**
- Given: 동접자 수십명의 사용자가 매일 오후 4시 동시에 PDF "다운로드" 버튼 연타
- When: `POST /generate` API가 무수히 백엔드를 때림
- Then: 이 응답 코드는 PDF 파일이 아직 구워지지 않았음에도 Fast Response(단말당 약 50ms, HTTP 202 Accepted 처리)로 즉시 반환되므로 커넥션 풀을 소모하지 않고 서버 다운을 방어한다.

**Scenario 2: 외부인 URL 유출 기한 (Presigned URL 보안)**
- Given: 발급된 `download_url` 문자열을 직원이 고의로 사원방 단체 카카오톡에 살포
- When: 수십 분 뒤에 퇴사자(외부인)가 그 링크를 통해 브라우저로 접근을 도모함
- Then: S3 Presigned URL 서명 규칙(ExpiresIn=300초 강제 제약)에 의해 토큰이 기한 만료되어 Access Denied 403 에러가 반환되어 기업 기밀이 지켜진다.

## ⚙️ Technical Constraints
- PDF를 만들고 S3에 밀어넣는 `async_generate_and_upload_pdf` 백그라운드 태스크는 트랜잭션 에러 발생 시, 처리 테이블(`DB-008`)의 Status를 반드시 `failed` 로 업데이트 하여 클라이언트가 무한 로딩 바에 갇히지 않도록 종결해야 한다.

## 🏁 Definition of Done (DoD)
- [ ] `BackgroundTasks` 파라미터를 활용한 202 Accepted 딜레이리스 응답 구현?
- [ ] S3/Supabase Storage를 상정한 `generate_presigned_url` 비즈니스 룰 고려?
- [ ] 프론트엔드가 대기(Polling) 할 수 있는 Status 체커 GET 함수 포함 여부?
