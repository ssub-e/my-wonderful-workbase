---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Adapter] ADAPT-005: Gemini LLM 연동 어댑터 (XAI 해설 산출)"
labels: 'adapter, external-api, phase:2, priority:high'
assignees: ''
type: task
tags: [task, integration]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [ADAPT-005] Google Gemini API 통신 객체 구성
- 목적: REQ-FUNC-006(자연어 기반 XAI 리포트)의 핵심으로, Python 기반의 통계 수치 및 SHAP Factor 요인들을 입력받아 현업 비전문가도 쉽게 이해하는 B2B 자연어 평문 해설(글)을 반환하는 프록시 어댑터를 구성한다.

## 🔗 References (Spec & Context)
- SRS 제약 스펙: [`SRS-V1.0.md#§1.5.3`](raw/assets/SRS-V1.0.md) — C-TEC-009 (Gemini API 의무 사용)
- XAI 기능 연관: [`SRS-V1.0.md#§4.1.1`](raw/assets/SRS-V1.0.md) — REQ-FUNC-006 (결과 해석 자연어 생성)

## ✅ Task Breakdown (실행 계획)

### 1. Google GenAI 패키지 연동
- [ ] `google-genai` 또는 적법한 http 래퍼 모듈 설치. (의도된 C-TEC-009 구조 보장 위함)
- [ ] `app/adapters/llm/gemini_adapter.py` 작성:
  ```python
  import google.generativeai as genai
  from app.core.config import settings

  class GeminiLLMAdapter:
      def __init__(self):
          # 설정 주입
          genai.configure(api_key=settings.GEMINI_API_KEY)
          self.model = genai.GenerativeModel('gemini-1.5-pro-latest') # 또는 flash 등 속도 최적화 버젼
          
      async def generate_explanation(self, prediction_qty: int, factors: List[Dict]) -> str:
          """
          factors = [{"name": "온도저하", "shap_weight": +3.2}, {"name": "휴일효과", "shap_weight": -1.5}]
          위 변수를 System Prompt에 버무려 LLM을 호출하고, 평문 해설 결과물 str만 얻어냄
          """
          prompt = self._build_prompt(prediction_qty, factors)
          # async 호출 대용 (패키지에 따라 thread pool 등 활용 방안 고려)
          response = await self.model.generate_content_async(prompt)
          return response.text
  ```

### 2. B2B 프롬프트 템플릿(System Prompt) 하드코딩 또는 관리
- [ ] `_build_prompt` 메서드 내부에 XAI를 설명하는 전문적인 프롬프트 어조(Persona) 확립.
  - *예시: "당신은 물류 예측 AI 어시스턴트입니다. 다음 SHAP 가중치 표를 분석하여 '왜 이만큼의 발주량이 나왔는지' 센터장님이 이해하기 쉽게 2~3문장 이내로 브리핑하세요. 환각(Hallucination)을 금지합니다."*

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: Factor To Text 파이프라인 (환각 차단)**
- Given: `{qty: 1500, shap_list: [(온도 -2도: +25% 비중), (주말: -10% 비중)]}` 인풋 데이터 셋
- When: 어댑터의 `generate_explanation` 호출
- Then: 외부 지식이 아닌 인풋 데이터를 근거로 해석한 평문("내일 온도가 2도 떨어짐에 따라 전일 대비 발주량이 증가할 것으로 보이며...")이 3초 이내에 `string` 으로 반환된다.

**Scenario 2: API 단절 또는 할당량(Quota) 소진 방어**
- Given: Gemini 토큰 한도 초과 오류 상태 (429 등 에러 전파됨)
- When: 서비스 로직 단 발화
- Then: 이 에러가 전체 화면 로딩을 깨뜨리지 않고 Catch 하여 `"시스템 오류로 AI 분석망을 잠시 불러올 수 없습니다."` 와 같은 Fallback 텍스트로 치환하여 서빙(앱 신뢰성 보호).

## ⚙️ Technical Constraints
- 동기(Sync) 호출 방식인 기본 생성기를 패스트 API 엔드포인트 로직에 넣을 경우 Event Loop이 멈춰 전체 502/Timeout 참사가 난다. 반드시 `generate_content_async` 를 사용하거나 `run_in_threadpool` 에 묶어 비동기(Async) 성향을 지켜야 한다 (FastAPI 룰).

## 🏁 Definition of Done (DoD)
- [ ] Gemini 패키지 설치 의존성 통과 및 `generate_content_async` 적용 여부?
- [ ] 도메인 지식 파라미터를 받을 수 있도록 프롬프트 파서 함수가 작성되었는가?
- [ ] 텍스트 문자열(string)을 클린하게 반환하는 반환 타입 검증 완료?
