---
name: skillers-finder
description: 스폰지클럽 이기적인 스킬러스 채널 멤버용 personalized 스킬 추천 + 자동 설치 + 슬랙 후기 템플릿 도구. 사용자의 현재 작업이나 빈 상태 인터뷰를 통해 레딧·깃헙·공식 마켓에서 가장 적합한 Claude Code 스킬 8~10개를 찾아 추천한다. "이 스킬 쓸게" 답하면 자동 다운로드. 그 다음 이기적인 스킬러스 채널에 공유할 후기 템플릿까지 작성. "스킬 추천해줘", "이런 거 만드는데 어떤 스킬?", "랜딩페이지 만들어줘", "회의록 정리 스킬", "어떤 스킬 깔지", "skillers-finder", "스킬러스 추천", "내 프로젝트에 맞는 스킬" 등 트리거.
license: MIT
---

# Skillers Finder v3 — 진짜 검색 기반 추천

스폰지클럽 이기적인 스킬러스 채널용. **레딧·깃헙 실제 검색** → 추천 8~10 → 자동 설치 → 채널 후기 템플릿.

---

## ⚠️ 안티패턴 (절대 하지 말 것)

이전 v1·v2의 named failure mode:
1. **WebSearch 안 돌리고** 일반 지식 + 하드코딩된 직무 우선순위 리스트로 추천 만들기
2. Anthropic 공식 스킬에 편향
3. "이번엔 못 찾았어" 라며 `~/.claude/skills/` 이미 깔린 거나 리스트해버림
4. 8~10개라 했는데 3~5개만 출력
5. **직무 라벨로 일반화** — "마케터 = canvas-design, viral-content" 같은 사전 매핑 사용. 같은 직무라도 사람마다 상황·프로젝트가 다른데 일반화하면 추천이 둔해짐

위 5개 중 **1개라도 하면 출력 무효**. 처음부터 다시.

---

## STEP 0 — 도구 로드 (MANDATORY)

WebSearch는 **deferred tool**임. 첫 번째 동작은 무조건:

```
ToolSearch select:WebSearch
```

WebSearch 로드 안 하고 추천 만들면 자동 실패. 사용자가 "안 돌아갔잖아" 알게 됨.

---

## STEP 1 — Mode 판별

사용자 입력에 작업 컨텍스트 명시?
- "랜딩 페이지 만들어줘" / "보고서 자동화" / "카드뉴스" 등 → **Mode A**
- "스킬 추천해줘" / "뭐 깔지 모르겠어" → **Mode B (인터뷰)**

---

## STEP 1A — Mode A 작업 + 프로젝트 컨텍스트 종합

### 1A-1. 작업 1줄 요약 + 확인
> "OK, **{사용자 발화 그대로}**에 쓸 스킬, 레딧 + 깃헙 직접 뒤져서 8~10개 정리할게요. 1~2분."

### 1A-2. 프로젝트 컨텍스트 자동 스캔 (핵심)

직무 라벨로 일반화 X. **사용자 지금 상황을 그대로 흡수**한다.

```bash
ls -la                    # cwd 무엇이 있나
cat CLAUDE.md 2>/dev/null   # 있으면 5초 안에 훑기
cat README.md 2>/dev/null   # 있으면 동일
ls .claude/skills/ 2>/dev/null  # 이미 깔린 스킬
git log --oneline -10 2>/dev/null  # 최근 작업 흐름
```

각 결과에서 추출:
- **프로젝트 타입**: 마크다운 vault / Next.js 사이트 / Python 분석 / Slack 봇 / 운영 폴더 등
- **현재 작업 흐름**: 최근 커밋·변경 파일에서 보이는 트렌드
- **이미 사용 중인 스킬·도구**: 중복 추천 방지
- **고유 컨텍스트 단어**: 브랜드명·용어·도메인 (예: "스폰지클럽", "fullstack-mkt-skills", "Remotion")

### 1A-3. 검색 키워드 합성 (사용자 발화 + 프로젝트 단서)

**일반 직무 키워드 X.** 사용자가 말한 작업 동사·명사 + 프로젝트 컨텍스트에서 나온 고유 단어 결합:
- 예: 사용자 "랜딩 페이지 만들어줘" + cwd = `emily-brand/site/` → 키워드: `landing page Next.js`, `brand site Vercel`, `personal brand landing`
- 예: 사용자 "회의록 정리" + cwd = Slack 봇 폴더 → 키워드: `meeting notes Slack bot`, `summarize transcript`, `Slack workflow`

키워드는 **사용자 작업 + 프로젝트 환경**에서 합성. 일반 "직무" 사전 매핑 X.

→ STEP 2

## STEP 1B — Mode B 인터뷰 (상황 기반, 직무 무관)

직무 질문은 **보조 정보**일 뿐. 메인은 사용자가 **지금 뭘 하고 있는지 + 어디서 막히는지**.

질문 1: "**지금 뭐 하고 있어요?** 프로젝트 한 줄 또는 매주 반복하는 작업 1-2개." — 메인 컨텍스트.

질문 2: "**그 작업에서 어떤 부분이 가장 답답해요?**" (시간 잡아먹기 / 매번 같은 거 설명 / 결과물 일관성 / 반복 검토 등)

질문 3: "**이미 사용 중인 도구·스킬 있나요?**" (중복 추천 방지)

질문 4 (선택): "**결과물 형태는?**" (PNG / PDF / 마크다운 / 코드 / 슬랙 메시지 등)

질문 5 (선택, AI 처음일 때만): "AI 도구 경험 — 처음/조금/많이?"

답변 받은 후 **1A-3과 동일하게 키워드 합성**. 사용자 답변에서 나온 동사·명사 + (있으면) 프로젝트 단서 결합.

→ STEP 2

---

## STEP 2 — 실제 검색 (MANDATORY, 최소 5회)

이 단계는 **NEVER SKIP**. 안 돌리면 STEP 3 출력 자체가 위반.

### 2-1. 실행할 5개 쿼리 (병렬 권장)

각 키워드별로 최소 1번씩 + 직무 맥락 1번 + 깃헙 1번 = 최소 **5 WebSearch 콜**:

```
1. WebSearch("site:reddit.com r/ClaudeAI {keyword1} skill")
2. WebSearch("site:reddit.com r/ClaudeCode {keyword2}")
3. WebSearch("{keyword1} claude skill SKILL.md github.com")
4. WebSearch("{keyword2} {keyword3} claude code skill 2026")
5. WebSearch("site:reddit.com r/{직무관련sub} claude skill viral")
```

직무별 추가 sub:
- 마케터·콘텐츠 → `r/marketing`, `r/SaaS`, `r/Entrepreneur`, `r/PersonalBrand`
- 디자이너 → `r/Design`, `r/UI_Design`, `r/graphic_design`
- 기획·PM → `r/ProductManagement`, `r/SaaS`
- 개발자 → `r/programming`, `r/webdev`, `r/LocalLLaMA`

### 2-2. 본인 환경 단서 (보조, 메인 X)

- `ls ~/.claude/skills/` 와 `ls .claude/skills/` (있으면)
- 이미 깔린 거 중 매칭되는 거 1-2개 추천에 포함 가능

### 2-3. 검증된 큐레이션 리스트도 1번 더 검색

```
6. WebSearch("awesome claude skills github.com 2026")
```

→ ComposioHQ/awesome-claude-skills, travisvn/awesome-claude-skills, minhnv0807/fullstack-mkt-skills 같은 권위 있는 컬렉션 발견.

### 2-4. 자체 체크 (출력 전 필수)

다음 스스로 확인. 하나라도 NO면 STEP 3 못 감:
- [ ] WebSearch 호출 ≥ 5회?
- [ ] 결과에 실제 깃헙 별·레딧 업보트 숫자 있나?
- [ ] 레딧/깃헙 출처 ≥ 70%?

NO면 → STEP 2 다시.

---

## STEP 3 — 추천 (8~10개 출력)

### 3-1. Diversity 룰 (필수)

8~10개 안에 다음 비율:
- **레딧·커뮤니티/유저 깃헙: 5~7개** (가장 큼)
- 공식 (Anthropic): 1~2개
- 본인 이미 설치한 거: 0~2개 (있으면)
- 큐레이션 리스트의 핫 컬렉션: 1~2개

**공식만 5개 이상이면 위반**. 즉시 다시 검색.

### 3-2. 출력 헤더

```
🐚 {작업}에 쓸 만한 스킬 N개 — 진짜 레딧·깃헙 검색 결과

검색 쿼리 N회 실행. 별 N+/업보트 N+ 기준 필터링.
출처 비율: 레딧·커뮤니티 N / 공식 N / 본인 환경 N / 큐레이션 N
```

### 3-3. 각 스킬 (7~12줄)

```
🐚 [N]. {스킬 이름} ⭐{star}  ({공식/커뮤니티/레딧 hot/본인 이미 설치})

**무엇**: {1~2줄}

**왜 너한테 좋은지**
{사용자 작업·직무와 직접 매핑되는 이유 + 시간/효과 구체 수치}

**어떤 작업에 좋은지**
- {활용 사례 1}
- {활용 사례 2}

**실제 신호 (검색 결과에서)**
> "{레딧 댓글 / 트윗 / GitHub README 인용}" — @author, {N upvotes / stars}
또는: "스타 N개, 레딧 r/sub에서 N번 언급"

**어떻게 시작?**
설치: `git clone {URL}` → `cp -r ... ~/.claude/skills/`
또는 Claude.ai 웹: .skill 파일 다운 → Settings → Skills → "+"

첫 명령: "{사용자 작업에 맞는 구체적 한 마디}"

**링크**: {GitHub URL}
```

### 3-4. 정렬

매칭도 + 별·업보트 + 진입 장벽 종합 → 1번이 가장 강력.

---

## STEP 4 — 사용자 픽 + 자동 설치

### 4-1. 출력 끝에 멘트

```
어떤 거 깔아볼래?
- "{이름} 쓸게" / "1번 쓸게" / "1번 3번 깔아줘"
- "한 개 골라줘" → 가장 매칭 높은 거 자동 픽
- "다른 거 더" → 추가 검색
```

### 4-2. 사용자 픽 → 자동 설치

```bash
# Anthropic 공식 스킬:
git clone --depth 1 https://github.com/anthropics/skills.git /tmp/anthropic-skills 2>/dev/null
cp -r /tmp/anthropic-skills/skills/{name} ~/.claude/skills/

# 커뮤니티 스킬:
mkdir -p ~/.claude/skills/{name}
git clone --depth 1 {repo_url} /tmp/{name}
# 레포 구조에 따라 적절한 폴더 복사
cp -r /tmp/{name}/skills/{specific}/* ~/.claude/skills/{name}/
# 또는 단일 SKILL.md:
curl -fsSL {raw_url} -o ~/.claude/skills/{name}/SKILL.md
```

설치 완료 멘트:
```
✅ {name} 설치 (~/.claude/skills/{name}/)

다음 세션부터 자동 발동.

지금 바로 써볼 명령:
{사용자 작업에 맞는 구체 명령 1줄}

결과 나오면 채널에 후기 올리실 거예요?
```

---

## STEP 5 — 슬랙 후기 템플릿 제안

설치 직후 무조건 묻기:

```
🐚 이거 이기적인 스킬러스 채널에 후기 공유할까요?

다른 멤버들도 이 스킬 알게 되면 도움 됩니다.
+1🐚 (단순 공유) 또는 +3🐚 (직접 써본 후기)

답: "OK / 작성해줘 / 그래" → 템플릿 자동 작성
답: "나중에 / X" → 스킵
```

---

## STEP 6 — 후기 템플릿 자동 작성 (사용자 OK 시)

이미 알고 있는 정보 자동 채움. 부족한 거 1개만 짧게 묻기:
> "지금 한 번 써보고 어땠어요? 1줄로. (시간 절약, 단점, 의외 발견)"

답변 받으면 다음 형식 그대로 출력:

```
[`{skill_name}_써본_후기`]

📌 한줄 요약
- {1줄 — 이 스킬이 뭐 하고 누구한테 좋은지}

🔍 주요 내용
- 박힌 능력 1
- 박힌 능력 2
- 출처: {공식 / 커뮤니티}, 별 N개, 링크: {GitHub URL}

💼 내가 써본 상황 + 결과
- 어떤 상황에서: {사용자 작업}
- 어떻게 썼는지: {입력 명령 1줄}
- 결과 / 인사이트: {시간 단축 + 단점·팁 + 사용자 답변}

🔗 링크 / 스크린샷
- 스킬: {GitHub URL}
- 결과물: (사용자 직접 첨부)
```

출력 끝에:
```
✅ 후기 템플릿 작성. 슬랙 #이기적인_스킬러스에 그대로 복붙.
   결과물(스크린샷) 첨부하면 +3🐚 자동 지급.
```

다른 멤버가 이 글 보고 즉시 따라할 수 있게 — **추상 표현 X, 구체 수치·명령·링크 박힘**.

---

## 톤 룰

- 평어, 군더더기 X
- 슬랙 코드블록(\`\`\`)으로 후기 템플릿 감싸 복붙 가능하게
- 이모지 🐚 외 절제
- "효율을 높입니다" 같은 추상 X → "매주 30분 절약" 구체

## 키워드 합성 원칙 (참고용)

직무별 사전 매핑 사용 X. 키워드는 매번 다음 **세 가지 source** 합성으로 만든다:

1. **사용자 발화** — 그대로 추출한 동사·명사
2. **인터뷰 답변** — 사용자가 말한 작업·고통 지점
3. **프로젝트 컨텍스트** — `ls`, `CLAUDE.md`, `README.md`, `.claude/skills/`, 최근 git log에서 나온 고유 단어

**같은 직무라도 매번 다른 키워드**가 나와야 정상. 마케터 A는 "Slack workflow + 보고서 자동화", 마케터 B는 "Personal brand + AI Avatar". 직무 라벨로 둘을 묶으면 추천이 둔해짐.

## 발동 조건

- "스킬 추천해줘" / "어떤 스킬 깔지"
- "{작업} 만들어줘" + 스킬 컨텍스트
- "skillers-finder" / "스킬러스 추천"
- "내가 쓸 만한 스킬"

---

## 자체 체크리스트 (출력 전 마지막 검증)

다음 모두 YES여야 출력 가능:

- [ ] WebSearch ≥ 5회 실제 호출했나?
- [ ] 추천 8~10개 (5개 이하면 위반)
- [ ] 레딧/커뮤니티 비율 50%↑
- [ ] 각 스킬에 별 수 또는 업보트 또는 인용 1개씩 있나?
- [ ] 사용자 작업과 매칭 명시 (추상 X)?
- [ ] 설치 명령 + 첫 사용 명령 둘 다 있나?

NO 1개라도 → STEP 2부터 다시.
