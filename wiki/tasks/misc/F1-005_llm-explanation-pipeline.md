---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Engine] F1-005: Gemini LLM 자동 자연어 해설 생성 파이프라인"
labels: 'forecast, xai, llm, priority:high, phase:2'
assignees: ''
type: task
tags: [task, misc]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [F1-005] 도출 예측값과 SHAP 요인들을 LLM이 평문으로 요약 번역
- 목적: REQ-FUNC-006(결과 해석 자연어 생성) 이행. F1-004에서 추출한 복잡한 기호(`{temp: -12.4}`)들을 일반 B2B 시스템 관리자(센터장, MD)가 이해 가능한 텍스트 문구로 치환한다. `ADAPT-005` (Gemini 어댑터)에 1일 단위 프롬프트를 주입하여 반환 받고 DB 내(`FORECAST_FACTOR` 의 `explanation_text` 등)에 밀어 넣는다.

## 🔗 References (Spec & Context)
- XAI 요건: [`SRS-V1.0.md#§4.1.1`](file:///e:/workspace/SRS-from-PRD/SRS-V1.0.md) — REQ-FUNC-006
- 의존 태스크: `ADAPT-005` (Gemini API 통신단), `F1-004` (SHAP 원인 분석 데이터소스)

## ✅ Task Breakdown (실행 계획)

### 1. LLM 통합 브릿지 설계
- [ ] `app/domain/forecast/engine/llm_describer.py`:
  ```python
  from app.adapters.llm.gemini_adapter import GeminiLLMAdapter
  import logging

  logger = logging.getLogger(__name__)

  class XAIDescriber:
      def __init__(self):
          self.adapter = GeminiLLMAdapter()
          
      async def generate_human_readable_report(self, sku_name: str, qty: int, shap_factors: list) -> str:
          # System / Instruction Prompt 조립
          context_text = f"상품 '{sku_name}'의 익일 예측 출고량은 {qty}건 입니다.\n"
          context_text += "이유가 되는 AI 모델의 수학적 기여도 스코어는 다음과 같습니다:\n"
          for f in shap_factors:
              context_text += f"- {f['name']}: {f['shap_val']}\n"
              
          instruction = (
              "\n당신은 SCM(공급망 관리) 전문가입니다. 위 수치들을 기반으로, "
              "왜 내일 저만큼의 출고가 이루어지는지 물류 센터장에게 브리핑하는 "
              "짧고 전문적인 브리핑 멘트를 2~3문장 이내로 작성하세요. 숫자 나열 사절."
          )
          
          try:
              # LLM External 추론 호출 (시간 소요)
              result = await self.adapter.generate_explanation(context_text + instruction)
              return result
          except Exception as e:
              logger.error(f"LLM Generation failed: {e}")
              return "시스템 오류로 해설 문구를 생성하지 못했습니다."
  ```

### 2. ETL 워커에 비동기 편입
- [ ] 추론 워커(F1-003)가 DB에 저장하기 전 마지막 단락에서 이 `generate_human_readable_report` 를 Await으로 거쳐 최종 LLM 문자열을 받고 `explanation_text` 칼럼에 반영한 뒤 커밋한다.

## 🧪 Acceptance Criteria (BDD/GWT)

**Scenario 1: 프롬프트 주입 및 자연어 해설 추출 성공**
- Given: 온도 저하(-10), 마스크 세일(+50) 이 잡힌 SHAP 로우 데이터 유입
- When: LLM 객체가 프롬프트를 빌드하여 다녀옴 
- Then: DB 내에 "최근 온도가 강하하는 추세와 더불어 어제 시작된 기획전 효과가 중첩되어 발주량이 단기 급증할 것으로 예측됩니다." 류의 문장이 저장되며 Dashboard 호출시 노출된다.

**Scenario 2: API 쿼터 소진 / 실패 방어 (Fail-safe)**
- Given: 설정된 Gemini 과금 토큰이 소진되어 403 에러 리턴
- When: Try/Except 구간 통과 
- Then: 원래 목적지인 출고 예측량(숫자) 적재를 롤백시키지 않고 문구만 단순 Fallback 스트링화("오류 발생") 처리 후 파이프라인 진행을 계속한다.

## ⚙️ Technical Constraints
- 대용량 SKU (수천 개 종류 상품) 전수 해설화는 API Limit 타격 비용이 매우 크다. *현 MVP/Phase 1 스펙 하에선, 샵(Shop) 별로 Top-tier 관심 상품/관심 거래건 (ex: 예측 물량이 평소 2배를 넘는 경우)에 한해서만 Conditional 룰로 발동시키게 if문을 두는 최적화를 제안한다.*

## 🏁 Definition of Done (DoD)
- [ ] System Instruction 구문이 포함된 프롬프트 조합기(`_build_prompt` 체계)가 구현됨?
- [ ] `generate_human_readable_report` 반환 문자가 Fallback 안전망(`except`)을 가지고 있는가?
- [ ] 비동기 `await` 호출 체계 점검?
