<div align="center">

# parallax

[![Claude Code](https://img.shields.io/badge/Claude_Code-skill-D97757)](#설치)
[![Codex](https://img.shields.io/badge/Codex-skill-412991)](#설치)
[![License](https://img.shields.io/github/license/kgyujin/parallax-skill)](LICENSE)

교차검증 스킬입니다. Claude와 Codex가 같은 대상을 따로 검토한 뒤 결과를 맞대어 봅니다.<br>
코드뿐 아니라 설계 문서, 화면, 이력서와 자기소개서, 데이터 분석, 글도 검토합니다.

</div>

한쪽 눈만으로는 거리를 잴 수 없듯, 같은 모델이 자기 결과물을 검토하면 만들 때 쓴 가정을 그대로 물려받습니다. 스스로 지어낸 문장이나 잘못 세운 전제를 스스로 의심하지 못합니다. parallax(시차)는 다른 모델이 같은 대상을 독립적으로 본 뒤 두 결과의 차이를 보여 줍니다.

## 결과를 읽는 법

두 모델의 지적을 네 칸으로 나눕니다.

| | 상대도 지적함 | 상대는 지적 안 함 |
| --- | --- | --- |
| **나도 지적함** | `AGREED` 믿을 만함 | `MINE_ONLY` 판정 필요 |
| **나는 지적 안 함** | `THEIRS_ONLY` 판정 필요 | 둘 다 못 본 영역 |

오른쪽 아래 칸도 보고서에 적습니다. 둘 다 못 본 영역이 있다는 걸 알아야 "교차검증했으니 안전하다"는 착각을 피할 수 있습니다.

## 검토 대상에 따라 기준을 바꿉니다

지금 폴더를 보고 알맞은 검토 기준(프로파일)을 고릅니다. 코드 리뷰 기준으로 자기소개서를 보면 엉뚱한 지적이 나오기 때문입니다.

| 프로파일 | 대상 | 알아보는 단서 |
| --- | --- | --- |
| `code` | 소스코드 변경 | `package.json`, `pyproject.toml`, `go.mod` |
| `frontend` | 화면 컴포넌트, 디자인 시스템 | `tailwind.config`, `components/`, `*.tsx` |
| `career` | 이력서, 자기소개서, 포트폴리오 | `이력서`, `자기소개서`, `공고`, `resume` |
| `design-doc` | 설계 문서, RFC, ADR, 기획서 | `docs/`, `adr/`, `*설계*.md` |
| `data` | 데이터 분석, 노트북 | `*.ipynb`, `data/`, `*.csv` |
| `writing` | 블로그, 노트, 문서 | `.obsidian/`, `_posts/`, `content/` |

프로파일마다 무엇을 결함으로 볼지, 어떤 증거를 요구할지, 무엇은 지적하지 않을지를 정해 둡니다. 예를 들어 `career`는 원본에 없는 경력, 수치, 기술(`fabrication`)을 항상 가장 심각한 문제로 봅니다. [job-application-tailor](https://github.com/kgyujin/job-application-tailor-skill)로 고친 서류를 이 프로파일로 검토하면 "작성 → 검증" 흐름이 됩니다.

## 설치

```bash
npx skills add kgyujin/parallax-skill -g
```

설치할 도구(Claude Code, Codex)를 고르는 화면이 나옵니다. Node.js가 필요합니다. 제 다른 스킬과 함께 고르려면 `npx skills add kgyujin/agent-skills -g`를 쓰세요.

| 하려는 일 | 명령 |
| --- | --- |
| 갱신 | `npx skills update -g parallax` |
| 삭제 | `npx skills remove -g parallax` |

교차검증에는 상대편 도구가 설치되고 로그인돼 있어야 합니다. Claude Code에서 쓰면 `codex --version`, Codex에서 쓰면 `claude --version`이 동작해야 합니다. 없으면 시작 단계에서 멈추고 "교차검증 미수행"으로 알립니다.

<details>
<summary>직접 설치(clone)</summary>

```bash
git clone https://github.com/kgyujin/parallax-skill.git ~/.claude/skills/parallax
git clone https://github.com/kgyujin/parallax-skill.git ~/.codex/skills/parallax
```

쓰는 도구 쪽 한 줄만 실행하면 됩니다. 폴더 이름이 호출 이름이 되므로 `parallax`로 받아야 `/parallax`로 부를 수 있습니다. 스킬 폴더를 인식하지 않는 Codex CLI라면 `~/.codex/AGENTS.md`에 다음을 넣습니다.

```markdown
## 교차검증
교차검증·2차 의견 요청이 오면 ~/.codex/skills/parallax/SKILL.md 의 절차를 따른다.
```

</details>

> [!NOTE]
> 한 컴퓨터에서는 npx 설치와 직접 설치 중 하나만 쓰세요. 두 방식이 같은 스킬 폴더를 씁니다.

## 사용법

```text
교차검증 해줘
```

```text
이 PR 코덱스랑 교차검증 돌려줘
```

```text
/parallax --profile career --mode deep

원본: ./이력서.md, ./자기소개서.md
수정본: ./자기소개서_회사_직무.md
공고: ./공고.txt
```

프로파일을 정하지 않으면 지금 폴더를 보고 고릅니다. 두 후보가 비슷할 때만 물어봅니다.

| 모드 | 진행 | 상대 호출 | 쓸 때 |
| --- | --- | --- | --- |
| `quick` | R0~R2 | 1번 | 가벼운 2차 의견 |
| `standard`(기본) | R0~R3 | 2번 | 보통 |
| `deep` | R0~R4 | 3번 | 되돌리기 어려운 변경, 제출 직전 서류 |

## 진행 방식

```text
R0  환경 확인, 프로파일 고르기, 상대에게 보낼 요청서 만들기
R1  따로 검토. 내 결과를 파일로 고정한 뒤에만 상대를 부른다
R2  네 칸 대조표 만들기
R3  의견이 다른 항목만 한 번 반박 → CONFIRMED / REJECTED / DISPUTED
R4  (deep) 고친 뒤 다시 검토
```

## 지키는 원칙

1. **먼저 내 결과를 고정합니다.** 상대 결과를 먼저 보면 뒤에 보는 쪽이 앞 결론에 끌려가 교차검증이 성립하지 않습니다.
2. **증거 없는 지적은 버립니다.** 모든 지적에는 위치와 다시 확인할 수 있는 실패 상황이 있어야 합니다. "더 좋게 쓸 수 있다"는 증거가 아닙니다.
3. **억지로 합의하지 않습니다.** 끝까지 의견이 다르면 `DISPUTED`로 양쪽 근거를 나란히 보여 줍니다. 이견도 결과입니다.
4. **상대는 읽기만 합니다.** 파일을 고치는 건 검증을 시작한 쪽뿐입니다.
5. **검증 못 한 것을 숨기지 않습니다.** 상대 호출이 실패하면 혼자 본 결과를 교차검증 통과로 보고하지 않습니다.

## 남는 기록

```text
.parallax-runs/<YYYYMMDD-HHMM-주제>/
├── brief.md              # 상대에게 보낸 요청 원문
├── findings-mine.json    # R1에서 고정한 내 결과
├── findings-theirs.json  # 상대 결과
├── session-id            # 상대 세션 번호(반박 라운드에서 이어 쓰기)
├── matrix.md             # 네 칸 대조표
├── adjudication.json     # R3 판정
└── report.md             # 최종 보고서
```

`brief.md`를 그대로 남기는 이유는 상대에게 무엇을 보냈는지 알아야 결과를 믿을 수 있어서입니다. 요청서에 내 결론이 섞였다면 그 검증은 독립적이지 않았던 것입니다.

## 폴더 구성

```text
parallax-skill/
├── SKILL.md                         # 교차검증 절차(Claude Code, Codex 공용)
├── profiles/                        # code, frontend, career, design-doc, data, writing, generic
├── reference/
│   ├── detection.md                 # 폴더를 보고 프로파일을 고르는 규칙
│   ├── protocol.md                  # R0~R4 상세, 실패 처리, 같은 지적 판정
│   └── schema/finding.schema.json   # 상대에게 요구하는 결과 형식
└── LICENSE
```
