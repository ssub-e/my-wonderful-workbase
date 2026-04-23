# LLM Wiki Schema & Conventions

이 문서는 이 프로젝트의 LLM 위키를 구축하고 유지보수하는 에이전트를 위한 규칙과 스키마입니다.

## 1. Directory Structure
- `raw/`: 사용자가 제공한 불변의(immutable) 소스 진실의 원천(Source of Truth). 에이전트는 이곳의 파일을 읽기만 하며, 절대 수정하지 않습니다.
- `wiki/sources/`: `raw/` 원본에 대한 LLM의 요약 및 발췌 문서.
- `wiki/concepts/`: 프로젝트의 핵심 비즈니스 로직, 아키텍처, 전략적 방향성.
- `wiki/entities/`: 데이터베이스 스키마, 페르소나, 핵심 객체 모델.
- `wiki/tasks/`: 개발 티켓, 스프린트 작업, 환경 설정 등 구체적 실행 단위.

## 2. Document Conventions
모든 `wiki/` 하위 마크다운 문서는 다음 규칙을 따릅니다:

### Frontmatter (YAML)
문서 최상단에 다음 메타데이터를 포함합니다.
```yaml
---
type: [source | concept | entity | task]
tags: [tag1, tag2]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

### Cross-references (Graph View Optimization)
Obsidian의 그래프 뷰 상호작용을 극대화하기 위해, 문서 최하단에 항상 연관된 문서들의 쌍방향 링크(Deep Link)를 `## Related` 섹션으로 제공해야 합니다.
```markdown
## Related
- [연관 개념](wiki/concepts/...)
- [출처 문서](wiki/sources/...)
```

## 3. Workflow
- **Ingest**: 새 원본이 들어오면 `wiki/sources`에 요약을 만들고, 파생되는 개념과 엔티티를 `concepts`, `entities`에 추가합니다. 마지막으로 `index.md`와 `log.md`를 갱신합니다.
- **Log**: `log.md`는 항상 최신 작업 내역을 최상단 또는 최하단에 이어 붙입니다(append-only).
- **Index**: `index.md`는 위키의 종합 카탈로그 역할을 하므로, 새 문서가 생성될 때마다 반드시 링크를 등록해야 합니다.
