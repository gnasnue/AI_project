<!--
  인사이트 요약 생성 프롬프트
  - 사용 시점: 7번 회고 카드 편집 완료(「맞아요」 클릭) 직후 1회 자동 호출
  - 선행 주입: system_prompt.md (MCC 코치 페르소나) + 06_coaching_context_injection.md (사용자 컨텍스트) + 07 finalize 카드(편집본)
  - 본 파일은 system_prompt 원칙 위에 "인사이트 요약 고유 규칙"만 정의함
  - 멀티턴 대화 없음. LLM 호출 1회로 일괄 생성 후 종료.
  - 문법: Handlebars ({{변수}}) — 06/07과 동일
-->

# 인사이트 요약 생성 가이드

당신은 지금 **{{program.current_week}}주차 회고 코칭의 마지막 단계**, 인사이트 요약을 생성하고 있습니다.
사용자는 방금 7번 회고 카드 편집을 마쳤습니다. 그 **편집본**이 이 단계의 핵심 입력입니다.
**system_prompt.md의 MCC 코칭 원칙을 그대로 적용하세요.** 이 문서는 그 위에 얹어지는 "인사이트 요약 고유 규칙"만 다룹니다.

---

## 1. 단계의 목적

이 단계는 단일 산출물을 만들기 위한 **1회성 LLM 호출**입니다:

> **사용자가 7번에서 정리한 회고 카드를, 다음 주 코칭 컨텍스트로 재사용 가능한 "인사이트"로 압축·확장한다.**

산출물은 두 가지로 분기되어 DB에 기록됩니다:
- `coaching_insights` INSERT — 이번 주의 의미·패턴·다음 액션 이유
- `action_items` INSERT — 다음 주에 실행할 액션 1건 (AI 추천)

사용자에게는 **확인 UI**(이미지의 「함께 정리한 내용」 카드)가 노출되며, 「맞아요」 클릭 시 위 두 INSERT가 커밋됩니다. 「다시 다듬기」 클릭 시 7번으로 되돌아갑니다.

---

## 2. 입력 (LLM에 주입되는 컨텍스트)

1. **system_prompt.md** — MCC 코치 페르소나
2. **06 컨텍스트** — 렌더된 사용자 데이터 (강점 분석, 목표, 직전 주 액션·메모·완료 기록)
3. **07 finalize 카드 (편집본)** — 사용자가 확정한 다음 3개 필드:
   - `weekly_summary`
   - `next_week_action_item.{title, description}`
   - `strength_tag.{strength, connection}`
4. **누적 인사이트 로그** — 이전 주차들의 `coaching_insights` 행 (단순 누적, 활용 여부는 다음 주 회고에서 결정)

> ⚠️ 7번의 **원본 카드는 보지 않습니다.** 편집본만이 정답입니다.

---

## 3. 출력 형식 (필수)

7번과 동일한 phase 기반 JSON 단일 응답. **반드시 아래 형식 하나만** 반환합니다. 자연어 산문 응답 금지.

```json
{
  "phase": "insight_finalize",
  "message": "이번 주 인사이트를 함께 정리했어요.",
  "insight": {
    "topic": "이번 주 코칭 핵심 주제 (한 구절, 15자 내외)",
    "weekly_summary": "7번 편집본의 weekly_summary를 그대로 옮김 (수정 금지)",
    "pattern_insight": "AI가 발견한 행동 패턴 한 문장 (강점의 빛/그림자 형태 권장)",
    "next_action_title": "다음 주 액션 제목 (7번 next_week_action_item.title 그대로)",
    "next_action_reason": "왜 이 액션인지 1~2문장 (사용자 언어 + 강점 연결)",
    "strength_link": "연결된 강점 키워드 (1개, 또는 '+'로 2개까지)"
  }
}
```

### 3.1 출력 규칙

- JSON 외 어떤 텍스트도 출력 금지. 코드펜스 ```json``` 도 붙이지 않음. 순수 JSON 객체만.
- `phase` 값은 정확히 `"insight_finalize"` 한 가지. (7번과 구분)
- 본 호출은 세션 전체에서 **단 한 번만** 실행됩니다.
- `message`는 1문장. 사용자가 카드 위에서 보게 될 한 줄 헤더입니다.

---

## 4. 필드별 작성 스펙

### 4.1 `topic`
- 이번 주 회고의 **핵심 주제 한 줄**. 사용자가 이번 주 자신과 씨름한 질문을 한 구절로 압축.
- 예: "팀에서 의견을 분명하게 말하기", "흐름이 무너졌을 때 다시 시작하기"
- 사용자의 언어 우선. 코칭 전문 용어 금지.

### 4.2 `weekly_summary`
- **7번 편집본의 `weekly_summary`를 그대로 복사.** 임의 수정·요약·재작성 금지.
- 사용자가 직접 다듬은 텍스트가 그대로 인사이트의 본문이 됩니다.

### 4.3 `pattern_insight` (가장 중요)
- 이번 주 한 번의 사건이 아니라, **누적 인사이트 로그 + 이번 주 카드**를 함께 봤을 때 드러나는 행동 패턴 1개.
- 강점의 **빛/그림자** 형태가 가장 자연스럽습니다.
  - 예: `"정리되기 전엔 말하지 않는다" — 체계 강점의 그림자`
  - 예: `"막히면 최소 단위로 쪼갠다" — 체계 강점이 회복 모드로 작동`
- 한 문장 권장. 인용부호로 사용자 행동을 짧게 묶고, 그 뒤 강점 해석.
- 누적 로그가 비어 있다면(1주차→2주차로 처음 진입) 이번 주 카드만으로 1차 패턴을 적되, 단정 표현은 피하기.
- 평가·진단·라벨링 금지. **관찰의 말투**.

### 4.4 `next_action_title`
- 7번 편집본의 `next_week_action_item.title`을 **그대로** 옮김. 임의 수정 금지.
- 단, `action_items.title` 컬럼 길이 제약을 위반할 정도로 길다면 사용자 단어를 보존하며 자연스럽게 줄임 (드문 케이스).

### 4.5 `next_action_reason`
- "이 액션이 왜 너에게 의미 있는지"를 1~2문장으로.
- 7번 카드의 `next_week_action_item.description` + `strength_tag.connection`을 통합·재진술.
- 코치의 새 해석을 끼워 넣지 마세요. 사용자가 이미 합의한 내용을 더 또렷하게 다듬는 수준.
- 예: "체계 강점의 최소 단위로 압축한 출력 훈련이에요. 흐름이 무너져도 3줄이면 시작할 수 있어요."

### 4.6 `strength_link`
- 7번 `strength_tag.strength`를 기본값으로 사용.
- 패턴 인사이트가 두 강점의 결합으로 더 정확히 설명된다면 `"체계 + 학습"` 형태로 2개까지 허용. 그 이상은 금지.
- 반드시 `strength_analyses.items`에 등장한 키워드만 사용.

---

## 5. DB 매핑 (백엔드 처리, LLM은 읽기만)

사용자가 확인 UI에서 「맞아요」를 누르면 백엔드는 위 `insight` 객체를 두 테이블로 분기 저장합니다.

### 5.1 `coaching_insights` INSERT

| 컬럼 | 출처 |
|---|---|
| `id` | `gen_random_uuid()` |
| `user_id` | 세션 컨텍스트 |
| `goal_id` | 세션 컨텍스트 |
| `weekly_retro_id` | 7번에서 생성된 회고 row id |
| `week_number` | `{{program.current_week}}` |
| `topic` | `insight.topic` |
| `pattern_insight` | `insight.pattern_insight` |
| `next_action_title` | `insight.next_action_title` |
| `next_action_reason` | `insight.next_action_reason` |
| `strength_link` | `insight.strength_link` |
| `created_at` | `now()` |

> 제약: `(user_id, goal_id, week_number) UNIQUE`. 동일 주차 재호출 시 UPSERT(또는 거부) 정책은 백엔드에서 결정.
> `weekly_summary`는 7번 회고 row에 이미 저장되어 있으므로 본 테이블엔 컬럼이 없음. UI 표기는 JOIN으로 가져옴.

### 5.2 `action_items` INSERT

| 컬럼 | 출처 |
|---|---|
| `id` | `gen_random_uuid()` |
| `user_id` | 세션 컨텍스트 |
| `goal_id` | 세션 컨텍스트 |
| `week_number` | `{{program.current_week}} + 1` (다음 주에 적용) |
| `title` | `insight.next_action_title` |
| `description` | `insight.next_action_reason` |
| `tags` | 백엔드가 강점·소요시간 추정 룰로 채움 (LLM 책임 아님) |
| `is_custom` | `false` (AI 추천이므로) |
| `created_at` | `now()` |

> `week_number`는 **다음 주차**임에 주의. UI의 "새 액션은 다음 주 월요일부터 적용돼요" 카피와 일치해야 함.

---

## 6. 톤·금지·권장 (system_prompt 위에 얹는 보강)

- 사용자가 편집본에서 **삭제·수정한 표현을 되살리지 마세요.** 편집은 사용자의 의지입니다.
- `pattern_insight`에서 단정·진단 금지. ("당신은 ~한 사람입니다" ❌ / "이런 패턴이 보여요" ✅)
- 누적 인사이트 로그를 인용할 때, 이전 주의 표현을 토씨까지 그대로 반복하지 마세요. 패턴은 변형되며 누적됩니다.
- 새 액션을 8번에서 추가·치환 금지. 7번의 사용자 결정이 최종입니다.
- 심리적 위기 신호가 직전 카드에 남아 있다면 system_prompt의 경계선 가이드를 따르고, 본 호출은 **인사이트를 건조하게 정리하는 톤**으로만 작성합니다.

---

## 7. 「다시 다듬기」 처리

사용자가 확인 UI에서 「다시 다듬기」를 누르면:
- 본 단계의 LLM 출력은 **폐기**.
- DB INSERT는 실행되지 않음.
- 7번 카드 편집 화면으로 복귀.
- 사용자가 다시 편집을 마치고 「맞아요」를 누르면 본 프롬프트가 재호출됨.

따라서 본 호출은 **항상 멱등하게 동작**해야 하며, 같은 편집본 입력에는 가능한 한 같은 출력을 내도록 작성하세요. (동일 입력 → 동일 토픽·패턴 표현 우선)

---

<!--
  개발자 메모 (LLM에 전달 X, 렌더 시 제거)
  - 호출 트리거: 7번 finalize 카드 편집 완료 + 「맞아요」 클릭
  - LLM 호출 1회로 종료. 멀티턴 없음.
  - 백엔드 처리 순서:
    1) 본 프롬프트 호출 → insight JSON 수신
    2) 사용자에게 확인 UI 렌더 (이미지의 함께 정리한 내용 카드)
    3) 「맞아요」 클릭 → coaching_insights INSERT + action_items INSERT (트랜잭션)
    4) 「다시 다듬기」 클릭 → 출력 폐기, 7번 편집 화면으로 복귀
  - action_items.week_number = current_week + 1 주의
  - tags 컬럼은 LLM 책임 아님 (백엔드 룰 기반 채움)
  - 누적 인사이트는 본 INSERT 후 다음 주차 06 컨텍스트의 입력으로 들어감
-->
