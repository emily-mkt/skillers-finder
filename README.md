# skillers-finder

스폰지클럽 이기적인 스킬러스 채널 멤버용 Claude Code 스킬.

**무엇**: 사용자의 현재 작업·프로젝트 컨텍스트에 맞춰 레딧·깃헙·공식 마켓에서 적합한 Claude Code 스킬 8~10개를 찾아 추천. "이거 쓸게" 답하면 자동 설치 + 슬랙 후기 템플릿까지 작성.

## 설치

### Claude Code (CLI)

한 줄로:
```bash
mkdir -p ~/.claude/skills/skillers-finder && \
  curl -fsSL https://raw.githubusercontent.com/emily-mkt/skillers-finder/main/SKILL.md \
  -o ~/.claude/skills/skillers-finder/SKILL.md
```

또는 클론:
```bash
git clone https://github.com/emily-mkt/skillers-finder.git /tmp/skillers-finder
mkdir -p ~/.claude/skills/skillers-finder
cp /tmp/skillers-finder/SKILL.md ~/.claude/skills/skillers-finder/
```

### Claude.ai 웹

1. 이 레포의 [SKILL.md](./SKILL.md) 다운로드
2. claude.ai → Settings → Capabilities → Skills → "+"
3. SKILL.md 업로드

## 사용

다음처럼 클로드한테 말하면 자동 발동:
- "스킬 추천해줘"
- "랜딩페이지 만들어줘" (작업 명시)
- "내 프로젝트에 맞는 스킬 찾아줘"
- "skillers-finder 써줘"

## 흐름

```
1. Mode 판별 (작업 명시 / 빈 상태 인터뷰)
2. 프로젝트 컨텍스트 자동 스캔 (cwd, CLAUDE.md, git log)
3. 키워드 합성 (사용자 발화 + 인터뷰 + 프로젝트)
4. 레딧·깃헙 최소 5회 검색
5. 추천 8~10개 (커뮤니티 5-7 / 공식 1-2 / 본인 환경 0-2)
6. "이거 쓸게" 답하면 자동 설치
7. 슬랙 #이기적인_스킬러스 후기 템플릿 자동 작성 (옵션)
```

## 핵심 원칙

- **직무 라벨로 일반화 X** — 같은 마케터라도 사람마다 상황·프로젝트가 다름. 매번 다른 키워드 → 매번 다른 추천.
- **레딧·커뮤니티 우선** — Anthropic 공식보다 viral한 커뮤니티 스킬을 더 많이 노출.
- **자동 설치 + 슬랙 후기까지 한 번에** — 시도 → 후기 → 채널 활성화 1사이클.

## 라이센스

MIT

## 만든이

Emily ([@emily-mkt](https://github.com/emily-mkt)) · 스폰지클럽 이기적인 스킬러스 유닛장
