# parallax-skill

![License](https://img.shields.io/github/license/kgyujin/parallax-skill)
![Last Commit](https://img.shields.io/github/last-commit/kgyujin/parallax-skill)
![Claude Code](https://img.shields.io/badge/Claude%20Code-compatible-6b46c1?logo=anthropic&logoColor=white)
![Codex](https://img.shields.io/badge/Codex-compatible-10a37f?logo=openai&logoColor=white)
![Skill Format](https://img.shields.io/badge/format-SKILL.md-blue)

**Claude와 Codex가 같은 대상을 각자 독립적으로 검토한 뒤 결과를 대조하는 양방향 교차검증 스킬.**

시차(視差). 한쪽 눈만으로는 거리를 잴 수 없습니다. 두 관측점의 차이가 있어야 깊이가 측정됩니다.

## 왜 필요한가

같은 모델이 자기 작업을 검토하면 **생성할 때 쓴 가정을 검토할 때도 그대로 물려받습니다.**
자기가 지어낸 문장, 자기가 놓친 경계 조건, 자기가 세운 잘못된 전제를 스스로 의심하지 못합니다.

다른 모델 계열이 같은 대상을 **독립적으로** 본 뒤 대조해야 그 가정이 드러납니다.
그래서 이 스킬의 산출물은 합의된 결함 목록이 아니라 **두 관측 사이의 차이**입니다.

|  | 상대 지적함 | 상대 지적 안 함 |
| --- | --- | --- |
| **내가 지적함** | `AGREED` — 신뢰도 높음 | `MINE_ONLY` — 판정 필요 |
| **내가 지적 안 함** | `THEIRS_ONLY` — 판정 필요 | **측정 불가 영역** |

오른쪽 아래 칸을 리포트에 명시하는 것도 설계에 포함돼 있습니다.
양쪽 다 안 본 영역이 드러나야 "교차검증했으니 안전"이라는 착각이 생기지 않습니다.

## 코드만 검증하는 스킬이 아닙니다

현재 경로를 읽어 **검증 렌즈를 자동으로 갈아끼웁니다.**
코드 리뷰 렌즈로 자기소개서를 보면 "변수명이…" 같은 소리가 나오기 때문입니다.

| 프로파일 | 대상 | 감지 신호 |
| --- | --- | --- |
| `code` | 소스코드 변경 | `package.json`, `pyproject.toml`, `go.mod` … |
| `frontend` | UI 컴포넌트 · 디자인 시스템 | `tailwind.config`, `components/`, `*.tsx` |
| `career` | 이력서 · 자기소개서 · 포트폴리오 | `이력서`, `자기소개서`, `공고`, `resume`, `portfolio` |
| `design-doc` | 설계 문서 · RFC · ADR · 기획 | `docs/`, `adr/`, `*설계*.md` |
| `data` | 데이터 분석 · 노트북 | `*.ipynb`, `data/`, `*.csv` |
| `writing` | 블로그 · 노트 · 문서 | `.obsidian/`, `_posts/`, `content/` |

각 프로파일은 **결함의 정의 · 증거 형식 · 카테고리 · 심각도 기준 · 지적 금지 항목**을 규정합니다.
마지막 항목이 특히 중요합니다 — 모델 둘이 붙으면 잔소리도 두 배가 되거든요.

### 예: `career` 프로파일

`fabrication`(원본에 근거 없는 경력·수치·기술)을 **무조건 critical**로 둡니다.
한 모델이 쓴 자기소개서를 같은 모델이 검토하면 자기가 지어낸 걸 못 잡습니다.
다른 모델이 **원본 서류와 한 줄씩 대조하며** 읽어야 잡힙니다.

[`job-application-tailor`](https://github.com/kgyujin/job-application-tailor-skill) 같은
서류 튜닝 스킬의 출력을 이 프로파일로 검증하면 생산 → 검증 파이프라인이 됩니다.

## 양방향 대칭

`SKILL.md` 파일 하나가 Claude Code와 Codex 양쪽에서 동일하게 동작합니다.
**오케스트레이터는 이 파일을 읽고 있는 쪽이고, 상대편은 나머지 한쪽입니다.**

| 실행 호스트 | 상대편 호출 |
| --- | --- |
| Claude Code | `codex exec -s read-only …` |
| Codex | `claude -p --output-format json …` |

어느 쪽에서 시작하든 교차검증이 성립합니다.

## 진행 방식

```
R0  환경 확인 → 프로파일 감지·표시 → 상대편 추론 강도 확인 → 브리프 생성
R1  블라인드 병렬. 내 findings를 파일로 고정한 다음에만 상대를 호출
R2  2x2 대조 매트릭스
R3  이견 항목만 1라운드 반박 → CONFIRMED / REJECTED / DISPUTED
R4  (deep 모드) 수정 후 재검증
```

| 모드 | 라운드 | 상대 호출 | 용도 |
| --- | --- | --- | --- |
| `quick` | R0–R2 | 1회 | 가벼운 2차 의견 |
| `standard` (기본) | R0–R3 | 2회 | 일반 |
| `deep` | R0–R4 | 3회 | 되돌리기 어려운 변경, 제출 직전 서류 |

## 핵심 원칙

1. **블라인드 우선** — 내 findings를 파일로 고정하기 전에는 상대 결과를 읽지 않습니다.
   순서를 어기면 뒤에 보는 쪽이 앞의 결론에 끌려가 교차검증이 성립하지 않습니다.
2. **증거 없는 지적은 폐기** — 모든 finding은 위치 + 재현 가능한 실패 시나리오를 갖습니다.
   "더 좋게 쓸 수 있다"는 증거가 아닙니다.
3. **합의를 강제하지 않음** — 이견이 남으면 `DISPUTED`로 양쪽 논거를 나란히 제시하고 끝냅니다.
   억지 수렴은 근거 없이 한쪽이 접히는 동조 붕괴를 부릅니다. **이견은 실패가 아니라 산출물입니다.**
4. **상대는 읽기만 함** — 파일 수정 권한은 오케스트레이터에게만 있습니다.
5. **미검증을 숨기지 않음** — 상대 호출이 실패하면 단독 검토 결과를 교차검증 통과로 보고하지 않습니다.

## 설치

### npx로 설치

clone 없이 설치하고, 갱신과 삭제도 `npx skills`로 합니다. Node.js가 필요합니다.

```bash
npx skills add kgyujin/parallax-skill -g -a claude-code -a codex   # 설치
npx skills update -g parallax                                      # 갱신
npx skills remove -g parallax                                      # 삭제
```

`-g`는 모든 프로젝트에서 쓰도록 사용자 폴더에 설치하고, `-a`는 설치할 도구를 고릅니다. `-a`를 빼면 설치할 때 물어봅니다. 같은 컴퓨터에서 아래 직접 설치와 함께 쓰지 마세요. 두 방식이 같은 스킬 폴더를 씁니다.

### Claude Code

```bash
git clone https://github.com/kgyujin/parallax-skill.git ~/.claude/skills/parallax
```

디렉터리명이 호출 이름이 됩니다. `parallax`로 클론해야 `/parallax`로 부를 수 있습니다.

### Codex

```bash
git clone https://github.com/kgyujin/parallax-skill.git ~/.codex/skills/parallax
```

스킬 폴더를 인식하지 않는 codex-cli 환경이라면 `~/.codex/AGENTS.md`에 한 줄 추가합니다.

```markdown
## 교차검증
교차검증·2차 의견 요청이 오면 ~/.codex/skills/parallax/SKILL.md 의 절차를 따른다.
```

### 사전 조건

- 상대편 CLI가 설치·인증되어 있어야 합니다 (`codex --version` 또는 `claude --version`)
- 없으면 R0에서 중단하고 "교차검증 미수행(미검증)"으로 보고합니다

## 사용 예시

```text
교차검증 해줘
```

```text
/parallax --profile career --mode deep

원본: ./이력서.md, ./자기소개서.md
수정본: ./자기소개서_카카오_백엔드.md
공고: ./공고.txt
```

```text
이 PR 코덱스랑 교차검증 돌려줘
```

프로파일을 지정하지 않으면 현재 경로에서 자동 감지합니다.
1위와 2위 점수가 접전이면 그때만 물어봅니다.

## 산출물

```text
.parallax-runs/<YYYYMMDD-HHMM-slug>/
├── brief.md              # 상대에게 보낸 원문 그대로
├── findings-mine.json    # R1에서 고정, 이후 불변
├── findings-theirs.json
├── session-id
├── matrix.md             # 2x2 대조표
├── adjudication.json     # R3 판정
└── report.md             # 최종 리포트
```

`brief.md`를 그대로 남기는 이유: 상대에게 무엇을 보냈는지 확인할 수 없으면
findings의 품질을 판단할 수 없습니다. 브리프에 내 결론이 섞여 들어갔다면
그 검증은 독립적이지 않았던 겁니다.

## 폴더 구조

```text
parallax-skill/
├── SKILL.md                          # 메인 프로토콜 (Claude Code / Codex 공용)
├── profiles/
│   ├── code.md  frontend.md  career.md
│   ├── design-doc.md  data.md  writing.md  generic.md
├── reference/
│   ├── detection.md                  # 경로 기반 프로파일 감지 규칙
│   ├── protocol.md                   # R0~R4 상세, 실패 처리, 매칭 판정
│   └── schema/finding.schema.json    # 상대에게 강제하는 출력 스키마
├── README.md
└── LICENSE
```

## 라이선스

[MIT](./LICENSE)
