# 02. 강점 리포트 PDF 업로드 AI 명세서
> CareerPT · AI 개입 포인트 #1-B
> 기반: Gallup CliftonStrengths 공식 리포트 PDF 파싱

---

## 1. 목적

사용자가 업로드한 **Gallup CliftonStrengths 공식 리포트 PDF**에서 Top 5 강점을 추출하고, [01_strength_interview.md](01_strength_interview.md)의 종료 응답과 **동일한 JSON 스키마**로 반환한다.
인터뷰 경로(p05)와 결과 화면(p06)이 동일하므로, 출력 포맷은 반드시 §6과 일치해야 한다.

---

## 2. 사용 맥락

| 항목 | 내용 |
|------|------|
| 진입 화면 | p04 (강점 진단 방식 선택) → "갤럽 결과지 업로드" 카드 선택 → 파일 업로드 UI |
| 결과 화면 | p06 (강점 결과 — Top 5 카드) — 인터뷰 경로와 공유 |
| 선행 조건 | 사용자가 p04에서 "갤럽 결과지 업로드" 선택 + PDF 첨부 |
| 후행 연결 | top5 결과 → p07 커리어 인터뷰 컨텍스트로 전달 / `strength_analyses` DB INSERT |
| 허용 입력 | **공식 Gallup CliftonStrengths 리포트 PDF만** (한글/영문 모두) |

---

## 3. DB 연동

인터뷰 경로와 동일하게 `strength_analyses`에 INSERT한다. **source 구분 컬럼은 두지 않는다.**

### 읽기 (업로드 직전 주입)
```
profiles.nickname        → message 문구에 사용
profiles.job_field       → (선택) summary 톤 조정용
profiles.career_level    → (선택) summary 톤 조정용
```

### 쓰기 (파싱 성공 시 INSERT)
```
strength_analyses
  └ user_id
  └ top5: JSON          → §6 출력 포맷의 top5 배열 전체
  └ summary: TEXT       → 한 줄 요약 (PDF의 개인화 설명 기반)
  └ confidence: ENUM    → 항상 'high' 고정
  └ created_at
```

재업로드 시 `user_id` 기준 upsert (인터뷰 경로 결과를 덮어쓰는 것도 허용).

---

## 4. AI 역할 및 원칙

### 역할
사용자의 갤럽 공식 리포트 PDF를 읽고, **Top 5 강점**과 각 강점의 **근거(evidence)**를 PDF의 개인화된 설명 문장에서 추출한다.

### 핵심 원칙
- **공식 리포트가 아닐 경우 절대 추측하지 않는다.** `valid: false`로 명확히 거부한다.
- Top 5 강점명은 PDF 표기 그대로 식별하되, 출력은 §7 강점 레퍼런스의 정식 명칭(`name_ko`, `name_en`, `domain`)으로 정규화한다.
- `evidence`는 PDF의 개인화 설명 문단에서 **사용자에게 가장 특징적으로 들어맞는 문장 1개**를 그대로 인용한다 (요약/창작 금지).
- `description`은 [01_strength_interview.md](01_strength_interview.md) §7 레퍼런스 문장을 앱 톤(2인칭 "~해요" 체)으로 변환하여 작성한다. PDF의 표현이 아닌 **레퍼런스 기반**이어야 두 경로의 결과 카드가 일관된다.
- `confidence`는 항상 `"high"` 고정.

---

## 5. 처리 흐름

### 5-1. 클라이언트 측
1. p04에서 "갤럽 결과지 업로드" 카드 클릭
2. 파일 선택 다이얼로그 (accept: `application/pdf`, `image/*`)
3. 파일 검증 (확장자, 크기 ≤ 30MB)
4. PDF를 base64 인코딩
5. Anthropic API 호출 (§7)
6. 응답 파싱 → `valid: true`면 p06 이동, `valid: false`면 재업로드 안내 모달

### 5-2. 단일 호출 구조

인터뷰와 달리 **멀티턴이 아닌 1회 호출**로 종결한다.
입력: PDF document block + 짧은 user 텍스트
출력: §6 JSON (Top 5 + summary + confidence 또는 valid: false)

---

## 6. 출력 포맷 (JSON)

### 성공 응답 (공식 갤럽 리포트로 판정)
```json
{
  "valid": true,
  "message": "갤럽 리포트를 잘 받았어요. 결과를 정리해드릴게요.",
  "top5": [
    {
      "rank": 1,
      "name_ko": "전략",
      "name_en": "Strategic",
      "domain": "T",
      "description": "어떤 상황에서도 의미 있는 패턴을 빠르게 읽고, 앞으로 나아갈 길을 찾아내요. 복잡함 속에서도 최적의 경로를 그려내는 힘이 있어요.",
      "evidence": "PDF 개인화 설명에서 추출한 인용 문장"
    }
  ],
  "summary": "PDF 개인화 설명을 종합한 한 줄 요약",
  "confidence": "high"
}
```

### 실패 응답 (공식 리포트가 아니거나 식별 불가)
```json
{
  "valid": false,
  "reason": "not_gallup_report" | "top5_not_found" | "unreadable",
  "message": "갤럽 공식 CliftonStrengths 리포트로 보이지 않아요. 다른 파일을 업로드해 주세요."
}
```

### 스키마 호환성 (p06 랜딩)
p06은 `top5`, `summary`, `confidence` 필드만 사용한다. 인터뷰 경로의 `session_end: true` 응답과 동일한 키 구조를 유지하므로 **별도 분기 없이 동일 컴포넌트로 렌더링된다**.

| 필드 | 인터뷰 경로 | PDF 경로 |
|---|---|---|
| `top5` | 동일 스키마 | 동일 스키마 |
| `summary` | 대화 기반 생성 | PDF 개인화 설명 기반 생성 |
| `confidence` | high/medium/low | 항상 high |
| `message` | 클로징 멘트 | 업로드 확인 멘트 |
| `valid` | (없음) | 추가 필드 |
| `session_end` | true | (없음, 단일 호출이므로 불필요) |

---

## 7. API 파라미터

| 파라미터 | 값 | 이유 |
|---|---|---|
| `model` | `claude-sonnet-4-6` | PDF 레이아웃 이해 + 개인화 문장 추출에 충분 |
| `temperature` | `0.2` | 추출 작업이므로 결정론적 출력 우선 |
| `max_tokens` | `2000` | top5 5개 × evidence 인용 + summary 분량 |
| `system` | §8 시스템 프롬프트 | 1회성 주입 |
| `messages[0].content` | `[{type:"document", source:{type:"base64", media_type:"application/pdf", data:<base64>}}, {type:"text", text:"이 갤럽 리포트에서 Top 5 강점을 추출해 주세요."}]` | PDF document block + 지시 |

**Prompt Caching**: PDF 자체가 매번 다르므로 시스템 프롬프트만 캐싱.

**예상 비용 (Sonnet 4.6 기준)**
- 호출 1회 (PDF 평균 10페이지): 약 400~700원
- 온보딩 1회성 → 월 1,000 사용자 기준 약 50~70만원

---

## 8. 시스템 프롬프트

아래를 `system` 파라미터에 그대로 사용한다.

---

```
[역할 정의]
당신은 사용자가 업로드한 Gallup CliftonStrengths 공식 리포트 PDF를 읽고, Top 5 강점과 각 강점의 근거 인용문을 추출하는 분석가입니다.
당신은 추측하지 않습니다. PDF가 공식 갤럽 리포트가 아니거나 Top 5를 명확히 식별할 수 없으면 valid: false로 응답합니다.

[입력 검증]
다음을 모두 만족해야 valid: true로 응답합니다:
1. PDF에 "CliftonStrengths" 또는 "StrengthsFinder" 또는 "갤럽 강점" 표기가 있음
2. Top 5 강점이 순위와 함께 명확히 표기되어 있음 (1위~5위)
3. 각 강점에 대한 개인화된 설명 문단이 포함되어 있음

하나라도 충족되지 않으면 valid: false 및 reason 코드를 반환합니다:
- "not_gallup_report": 갤럽 리포트로 보이지 않음
- "top5_not_found": 갤럽 리포트로 보이나 Top 5 식별 불가
- "unreadable": PDF가 손상되었거나 OCR 불가

[Top 5 추출 규칙]
- PDF에 표기된 강점명을 아래 §강점 레퍼런스의 한글명/영문명과 매칭하여 정규화한다.
- 한글 리포트면 한글명, 영문 리포트면 영문명으로 매칭하되, 출력은 항상 두 언어 모두 채운다.
- domain은 레퍼런스의 도메인 약어를 그대로 사용한다 (E/I/R/T).

[evidence 작성 규칙]
- PDF의 해당 강점 개인화 설명 문단에서, 사용자의 특성을 가장 잘 드러내는 문장 1개를 그대로 인용한다.
- 길이는 한국어 기준 30~80자.
- 요약하거나 새로 쓰지 않는다. PDF에 있는 문장이어야 한다.
- 만약 PDF에 개인화 설명이 빈약하면 가장 가까운 문장을 인용하고, 그것도 없으면 빈 문자열로 둔다.

[description 작성 규칙]
- 아래 §강점 레퍼런스 문장을 앱 톤("~해요" 체, 2인칭)으로 변환하여 작성한다.
- PDF의 표현이 아닌 반드시 레퍼런스 기반이어야 한다 (인터뷰 경로 결과와 톤 일치를 위함).
- 길이는 80~140자.

[summary 작성 규칙]
- Top 5 강점의 조합이 사용자에게 어떤 의미를 가지는지 PDF 개인화 설명을 종합하여 한 줄로 요약한다.
- 길이는 한국어 기준 40~80자.
- 평가/조언 금지. 사실 기반 요약만.

[confidence]
- 항상 "high" 고정.

[강점 레퍼런스 — 갤럽 CliftonStrengths 34]
도메인 약어: E=실행력, I=영향력, R=관계구축, T=전략적사고

E 성취 ACHIEVER / I 행동 ACTIVATOR / R 적응 ADAPTABILITY / T 분석 ANALYTICAL
E 정리 ARRANGER / E 신념 BELIEF / I 주도력 COMMAND / I 커뮤니케이션 COMMUNICATION
I 승부 COMPETITION / R 연결성 CONNECTEDNESS / E 공정성 CONSISTENCY / T 회고 CONTEXT
E 심사숙고 DELIBERATIVE / R 개발 DEVELOPER / E 체계 DISCIPLINE / R 공감 EMPATHY
E 집중 FOCUS / T 미래지향 FUTURISTIC / R 화합 HARMONY / T 발상 IDEATION
R 포용 INCLUDER / R 개별화 INDIVIDUALIZATION / T 수집 INPUT / T 지적사고 INTELLECTION
T 배움 LEARNER / I 최상화 MAXIMIZER / R 긍정 POSITIVITY / R 절친 RELATOR
E 책임 RESPONSIBILITY / E 복구 RESTORATIVE / I 자기확신 SELF-ASSURANCE / I 존재감 SIGNIFICANCE
T 전략 STRATEGIC / I 사교성 WOO

(각 강점의 상세 description 원문은 01_strength_interview.md §8을 참조하여 동일하게 적용한다.)

[출력 규칙]
- 마크다운, 코드블록 래핑 없이 순수 JSON만 출력한다.
- valid: true일 때 top5는 반드시 5개, summary와 confidence: "high" 필수.
- valid: false일 때 reason과 message만 반환한다.
- top5의 name_ko, name_en, domain은 위 강점 레퍼런스에서 그대로 가져온다.

출력 형식:
{
  "valid": boolean,
  "message": "string",
  "top5": array | null,
  "summary": "string | null",
  "confidence": "high" | null,
  "reason": "not_gallup_report" | "top5_not_found" | "unreadable" | null
}
```

---

## 9. 프론트 연동 가이드

```javascript
// p04에서 "갤럽 결과지 업로드" 선택 → 파일 업로드 후 호출
async function uploadGallupReport(file, profile) {
  // 1. 검증
  if (file.size > 30 * 1024 * 1024) {
    showError('파일이 너무 큽니다 (30MB 이하).');
    return;
  }

  // 2. base64 인코딩
  const base64 = await fileToBase64(file);

  // 3. API 호출
  showLoading('갤럽 리포트 분석 중...');
  const res = await callAnthropicAPI({
    model: 'claude-sonnet-4-6',
    max_tokens: 2000,
    temperature: 0.2,
    system: buildSystemPrompt(),  // §8
    messages: [{
      role: 'user',
      content: [
        {
          type: 'document',
          source: {
            type: 'base64',
            media_type: 'application/pdf',
            data: base64
          }
        },
        {
          type: 'text',
          text: '이 갤럽 리포트에서 Top 5 강점을 추출해 주세요.'
        }
      ]
    }]
  });

  // 4. 파싱
  let data;
  try {
    data = JSON.parse(res.content[0].text);
  } catch (e) {
    showRetryModal('파일을 읽는 데 실패했어요. 다시 업로드해 주세요.');
    return;
  }

  // 5. 분기
  if (!data.valid) {
    const messages = {
      not_gallup_report: '갤럽 공식 CliftonStrengths 리포트로 보이지 않아요. 갤럽 사이트에서 받은 PDF를 업로드해 주세요.',
      top5_not_found: 'Top 5 강점을 찾지 못했어요. 전체 리포트(34개)가 아닌 Top 5 리포트인지 확인해 주세요.',
      unreadable: '파일을 읽지 못했어요. 다른 파일을 업로드해 주세요.'
    };
    showRetryModal(messages[data.reason] || data.message);
    return;
  }

  // 6. DB 저장 + p06 이동
  await saveToDB({
    user_id: profile.user_id,
    top5: data.top5,
    summary: data.summary,
    confidence: data.confidence  // 항상 'high'
  });
  nav('p06', data);  // 인터뷰 경로와 동일한 데이터 형태
}
```

---

## 10. 주의사항 및 엣지케이스

| 상황 | 대응 |
|------|------|
| 영문 리포트 업로드 | 정상 처리. name_ko/name_en 모두 채워서 출력 |
| 34개 전체 리포트 (Top 5만이 아닌) | Top 5 섹션을 우선 식별하여 추출. 실패 시 reason: "top5_not_found" |
| OCR이 필요한 스캔본 | Claude document input이 자동 처리. 품질 낮으면 reason: "unreadable" |
| 갤럽이 아닌 다른 강점 진단 (VIA, MBTI 등) | reason: "not_gallup_report" |
| JSON 파싱 오류 | 재시도 1회. 실패 시 재업로드 안내 |
| 인터뷰 결과가 이미 존재하는 사용자가 PDF 업로드 | upsert로 덮어쓰기 |

---

*CareerPT AI 명세서 v1.0 · 2026-05-05*
*기반: Gallup CliftonStrengths 공식 리포트 · 01_strength_interview.md와 출력 호환*
