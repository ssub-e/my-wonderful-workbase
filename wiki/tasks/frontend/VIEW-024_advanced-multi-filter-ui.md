---
name: Feature Task
about: SRS 기반의 구체적인 개발 태스크 명세
title: "[Feature] VIEW-024: 대용량 데이터 필터링을 위한 '고급 다중 선택(Multi-filter) 패널' UI"
labels: 'feature, frontend, ux, priority:medium'
assignees: ''
type: task
tags: [task, frontend]
created: 2026-04-22
---
## 🎯 Summary
- 기능명: [VIEW-024] 수천 개 SKU 및 카테고리 대응을 위한 고급 데이터 필터
- 목적: REQ-FUNC-012 확장. 수만 개의 SKU와 다양한 창고/카테고리를 보유한 엔터프라이즈 화주사가 특정 조건(예: '경기 지역' 창고의 'A브랜드' 중 '식품' 카테고리)의 데이터만 정교하게 필터링하여 대시보드에 반영할 수 있도록 다중 선택 및 검색이 결합된 검색 패널을 구축한다.

## 🔗 References (Spec & Context)
> 💡 AI Agent & Dev Note: 작업 시작 전 아래 문서를 반드시 먼저 Read/Evaluate 할 것.
- 디자인 시스템: `VIEW-002` (PopOver, Checkbox)
- 관련 API: `DASH-001`, `DASH-002` (쿼리 파라미터 연동)

## ✅ Task Breakdown (실행 계획)
- [ ] `FilterPanel` 컴포넌트 개발: 아코디언(Accordion) 또는 사이드바 형태로 구성
- [ ] **Multi-Select Combobox**: `shadcn/ui`의 Command 컴포넌트를 활용하여 수천 개의 SKU 중 검색 및 다중 선택 기능 구현
- [ ] **Hierarchical Category Filter**: 대/중/소 분류 트리 구조의 체크박스 필터 구현
- [ ] **Global State Sync**: 필터 선택 시 URL Query String(`?skus=1,2&cat=food`)과 동기화하여 '새로고침' 시에도 유지되도록 구현
- [ ] `react-query`의 `invalidateQueries`와 연동하여 필터 적용 즉시 대시보드 전체 차트 갱신 트리거
- [ ] 데이터가 너무 많을 경우를 대비한 '가상 스크롤(Virtual List)' 적용 (Combobox 내부)

## 🧪 Acceptance Criteria (BDD/GWT)
Scenario 1: 복합 필터 적용 및 차트 갱신
- Given: 대시보드 메인 화면
- When: 필터 패널에서 '카테고리: 신선식품', '창고: 경인센터'를 선택하고 '적용' 클릭
- Then: URL이 호출되고, 화면의 모든 KPI 카드(`VIEW-005`)와 시계열 차트(`VIEW-006`)가 해당 조건에 맞는 데이터로 즉시 리로드된다.

Scenario 2: 대량 SKU 검색 및 선택
- Given: 5,000개의 SKU를 모델링한 데이터셋
- When: SKU 필터 검색창에 '딸기' 입력
- Then: 초성 검색 및 부분 일치 검색이 작동하여 관련 SKU들만 리스트에 노출되며, '전체 선택' 기능이 정상 작동한다.

## ⚙️ Technical & Non-Functional Constraints
- UX: '초기화' 버튼을 통해 모든 필터를 한 번에 지우는 기능 포함
- 성능: 필터값이 바뀔 때마다 즉시 요청(Debounce) 혹은 '적용' 버튼 클릭 시 일괄 요청 중 비즈니스 요구사항에 따라 선택
- 신뢰성: 잘못된 필터 조합(예: 존재하지 않는 카테고리) 입력 시 토스트 알림으로 안내

## 💻 Implementation Snippet (Multi-Select Filter Logic)
```tsx
const SkuFilter = ({ options, selectedValues, onChange }: Props) => {
  const [search, setSearch] = useState("");
  const filteredOptions = useMemo(() => 
    options.filter(opt => opt.label.includes(search)).slice(0, 50), 
    [options, search]
  );

  return (
    <Command className="border rounded-lg">
      <CommandInput placeholder="Search SKU..." value={search} onValueChange={setSearch} />
      <CommandList>
        <CommandGroup>
          {filteredOptions.map((opt) => (
            <CommandItem key={opt.value} onSelect={() => onChange(opt.value)}>
              <Checkbox checked={selectedValues.includes(opt.value)} />
              <span className="ml-2">{opt.label}</span>
            </CommandItem>
          ))}
        </CommandGroup>
      </CommandList>
    </Command>
  );
};
```

## 🏁 Definition of Done (DoD)
- [ ] 모든 Acceptance Criteria를 충족하는가?
- [ ] URL Query String과 필터 상태가 완벽하게 양방향 바인딩(Two-way binding)되는가?
- [ ] 1,000개 이상의 필터 옵션 렌더링 시 브라우저 렉(Lag)이 없는가?

## 🚧 Dependencies & Blockers
- Depends on: `VIEW-002`, `DASH-001`
- Blocks: 없음
