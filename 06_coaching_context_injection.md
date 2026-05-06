<!--
  회고 코칭 컨텍스트 주입 템플릿
  - 사용 시점: 2주차 이상 회고 코칭 세션 시작 시
  - 1주차는 별도 흐름 (강점/커리어 인터뷰 결과만 사용) — 이 파일은 다루지 않음
  - 문법: Handlebars ({{변수}}, {{#each}}, {{#if}})
  - 시스템 프롬프트/코치 페르소나는 회고 코칭 .md에서 별도 처리
  - "이번 주" 기준: 직전 종료 주 (월~일). 예) 3주차 코칭 → 2주차 데이터.
-->

# 사용자 컨텍스트

현재 프로그램 진행 상황: **{{program.current_week}}주차 / 총 12주**
직전 종료 주차: **{{program.last_week}}주차** ({{program.last_week_start}} ~ {{program.last_week_end}})

---

## 1. 사용자 프로필 (베이스 — 항상 주입)

### 1.1 기본 프로필 (`profiles`)
- 이름: {{profile.name}}
- 나이: {{profile.age}}
- 직무/포지션: {{profile.role}}
- 경력: {{profile.experience}}
- 기타: {{profile.notes}}

### 1.2 강점 분석 (`strength_analyses`)
{{strength_analyses.summary}}

상세:
{{#each strength_analyses.items}}
- **{{this.title}}**: {{this.detail}}
{{/each}}

### 1.3 커리어 인터뷰 결과 (`career_interview_results`)
- 현재 상황: {{career_interview_results.current_state}}
- 지향 방향: {{career_interview_results.direction}}
- 주요 키워드: {{career_interview_results.keywords}}
- 도출된 역량 방향: {{career_interview_results.competency_direction}}

### 1.4 목표 히스토리 (`goals` — 전체)
{{#each goals_all}}
- [{{this.status}}] {{this.title}} ({{this.created_at}} ~ {{this.updated_at}})
  - 설명: {{this.description}}
{{/each}}

---

## 2. 직전 주({{program.last_week}}주차) 활동 데이터

### 2.1 직전 주 액션아이템 (`action_items`)
{{#each last_week_action_items}}
- **{{this.title}}**
  - 설명: {{this.description}}
  - 마감/주기: {{this.cadence}}
  - 생성일: {{this.created_at}}
{{/each}}

### 2.2 직전 주 액션 완료 기록 (`action_completions`)
{{#each last_week_action_completions}}
- {{this.completed_at}} — action_id: {{this.action_item_id}} ({{this.action_item_title}})
  - 상태: {{this.status}}
  - 메모: {{this.note}}
{{/each}}

### 2.3 직전 주 평일 메모 (`daily_memos` — 원문)
{{#each last_week_daily_memos}}
**{{this.date}} ({{this.weekday}})**
{{#if this.linked_action_item_title}}
이번 주 액션: {{this.linked_action_item_title}}
{{/if}}

{{this.content}}

---
{{/each}}

### 2.4 직전 주 회고 코칭 대화 로그 (`weekly_retros`)
> 직전 주 회고 코칭에서 사용자와 코치가 나눈 대화 전문.

{{#each last_week_retro_messages}}
**{{this.role}}** ({{this.created_at}}):
{{this.content}}

{{/each}}

---

## 3. 최근 3주 인사이트 요약 (`coaching_insights`)

> 3주차 이상부터 적용. 있는 만큼만 주입 (최대 3개, 최신순).

{{#if recent_coaching_insights.length}}
{{#each recent_coaching_insights}}
### {{this.week_number}}주차 인사이트 ({{this.created_at}})
{{this.summary}}

핵심 키워드: {{this.keywords}}

{{/each}}
{{else}}
_아직 누적된 인사이트 요약이 없습니다._
{{/if}}

---

<!--
  주입 규칙 메모 (개발자용 — LLM에 전달 X, 렌더 시 제거)
  - 모든 daily_memos / action_items / action_completions: 직전 주(월~일) 원문 그대로
  - goals: 전체 히스토리 (status 무관)
  - coaching_insights: created_at DESC, LIMIT 3 (없으면 섹션 비움)
  - weekly_retros: 직전 주차 1건의 대화 로그 전체
  - 1주차 코칭 시에는 이 템플릿을 호출하지 않음
-->
