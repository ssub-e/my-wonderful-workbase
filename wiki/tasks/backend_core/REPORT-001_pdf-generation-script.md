---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Report] REPORT-001: PDF 결재 문서 동적 생성 템플릿/스크립트"
labels: 'report, backend, phase:3, priority:low'
assignees: ''
type: task
tags: [task, backend_core]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [REPORT-001] HTML/CSS(Jinja2) 기반 PDF 문서 동적 생성 워커
- 목적: 센터장(B2B 관리자) 본사에 올려보낼 **예측 결과 + XAI 이유 + 적정인원 결재서류**를 웹 대시보드 밖에서도 열람 가능하도록 A4 PDF 포맷으로 만들어 준다 (REQ-FUNC-007 목적). C-TEC-007 에 따라 `WeasyPrint` (또는 FPDF2) 패키지를 파이프라인 연계에 얹어 구현한다.

## 🔗 References (Spec & Context)
- SRS 다운로드 요구: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-007 ("XAI 분석 결과 원클릭 다운로드")
- 기술 스택 제약: C-TEC-007 (`weasyprint` 우선 사용 권장 - 렌더링 품질 우수)
- 파일 저장 연관: `DB-008` (Report 기록 테이블)

## ✅ Task Breakdown (실행 계획)

### 1. Jinja2 기반 HTML/CSS 템플릿 생성
- [ ] `app/templates/report_template.html` 작성:
  ```html
  <html>
  <head>
      <style>
          @page { size: A4; margin: 20mm; }
          body { font-family: 'Malgun Gothic', 'Apple SD Gothic Neo', sans-serif; }
          .header { font-size: 24pt; font-weight: bold; border-bottom: 2px solid #000; }
          .xai-box { background-color: #f4f4f4; padding: 15px; margin-top:10px; }
      </style>
  </head>
  <body>
      <div class="header">{{ target_date }} {{ tenant_name }} 일일 발주/인력 결재서</div>
      <p><b>1. 익일 총 예측 박스 수:</b> {{ total_qty }} Box</p>
      <p><b>2. AI 분석 코멘트 (Gemini 도출):</b></p>
      <div class="xai-box">{{ xai_explanation_text }}</div>
      <p><b>3. 데이터 산출 적정인원:</b> {{ recommended_workers }} 명</p>
      <hr>
      <p style="text-align: right">본 문서는 AI 시스템에 의해 {{ created_at }} 자동 생성됨.</p>
  </body>
  </html>
  ```

### 2. WeasyPrint 렌더러 파이썬 브릿지
- [ ] `app/domain/report/services/pdf_generator.py` 구현:
  ```python
  import tempfile
  from weasyprint import HTML
  from jinja2 import Environment, FileSystemLoader

  def render_pdf_to_tempfile(context_data: dict) -> str:
      """
      context_data = {"target_date": "2026-05-01", "total_qty": 4500, ...}
      Jinja로 바인딩 후 WeasyPrint로 임시 디렉토리에 PDF를 구운 뒤 경로 반환
      """
      env = Environment(loader=FileSystemLoader("app/templates"))
      template = env.get_template("report_template.html")
      html_out = template.render(**context_data)
      
      # 임시파일 보관 체계 (삭제 등 안전관리 위함)
      temp_pdf = tempfile.NamedTemporaryFile(delete=False, suffix=".pdf")
      HTML(string=html_out).write_pdf(temp_pdf.name)
      
      return temp_pdf.name 
  ```

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 한글 폰트(UTF-8) 깨짐 없는 PDF 렌더링 보장**
- Given: Jinja Template에 유니코드 한글 XAI 문구 "전일 대비 강수량이 급증하여..." 가 주입된 모델
- When: `weasyprint.write_pdf` 가 컴파일됨
- Then: 도출된 PDF 파일을 OS 에서 열었을 때, 폰트 깨짐(ㅁㅁㅁ) 현상 없이 깔끔한 고도화된 한글 블럭이 인쇄 퀄리티(A4) 형태로 보존되어야 한다. (Docker 템플릿에 `fonts-nanum` 등 한글 폰트 사전 인스톨 필수 요구됨).

## ⚙️ Technical Constraints
- PDF 생성은 심각한 CPU 점유(Blocking)를 일으키는 무거운 작업이다. HTTP Request 안에 동기로 물려선 안 되며, `BackgroundTasks` 에 함수를 넘겨서 비동기 뒷단 생성(DB-008 status='processing') 되도록 짤 것.

## 🏁 Definition of Done (DoD)
- [ ] HTML/CSS 결재 문서 뼈대와 Jinja 변수 주입 문법 작성?
- [ ] Python에서 `Weasyprint` 컴파일 구문을 호출하는 함수 작성?
- [ ] 파일 입출력 로직(`tempfile`) 적용 및 사용 후 메모리 리크 회피 고려?
