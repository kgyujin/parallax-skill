# 프로토콜 상세

`SKILL.md`의 R0~R4를 실제로 실행할 때 필요한 절차·템플릿·판정 기준.

## run-id

```bash
RUN=".parallax-runs/$(date +%Y%m%d-%H%M)-<slug>"
mkdir -p "$RUN"
```

`<slug>`는 검증 대상을 알아볼 수 있는 짧은 이름 (`dialog-a11y`, `자소서-카카오`, `adr-007`).

## R0 — 브리프 템플릿

`$RUN/brief.md`에 아래 형식으로 쓴다. **내 판단·결론·의심 지점은 넣지 않는다.**

```markdown
# 교차검증 요청

당신은 독립 검토자입니다. 아래 대상을 직접 읽고 결함을 찾아주세요.
요청자의 판단은 의도적으로 제공하지 않았습니다. 당신의 관측이 필요합니다.

## 대상

<검증 대상 경로. 상대가 직접 열어볼 수 있게 경로로 준다>

## 작업 의도

<무엇을 하려던 작업인가. 1~3문장. 평가는 넣지 않는다>

## 프로파일: <이름>

### 결함의 정의
<profiles/<이름>.md 에서 그대로 옮긴다>

### 증거 형식 (필수)
<profiles/<이름>.md 에서 그대로 옮긴다>

### 카테고리
<표를 그대로 옮긴다>

### 심각도 기준
<그대로 옮긴다>

### 지적하지 말 것
<그대로 옮긴다. 이 목록이 노이즈를 막는 핵심이다>

## 제약

<AGENTS.md / CLAUDE.md / 규약 파일 경로. 있으면 반드시 안내>
<프로파일별 브리프 추가 항목>

## 출력

아래 JSON 스키마를 정확히 따르세요. 스키마 밖 필드를 추가하지 마세요.
증거를 채울 수 없는 지적은 내지 마세요 — 빈 findings 배열이 억지 findings보다 낫습니다.
검토하지 못한 범위는 not_reviewed에 정직하게 적어주세요.
```

## R1 — 순서 강제

```
1. profiles/<이름>.md 를 읽는다
2. 대상을 내가 직접 검토한다
3. $RUN/findings-mine.json 을 쓴다        ← 여기까지 끝나야
4. 상대를 호출한다                          ← 여기로 넘어간다
5. $RUN/findings-theirs.json 을 읽는다
```

**3번이 끝나기 전에 5번을 하면 이 스킬은 아무 가치가 없다.**
상대 결과를 먼저 본 상태에서 만든 내 findings는 독립 관측이 아니라 동의 표시다.

`findings-mine.json`은 저장 후 수정하지 않는다. 상대 결과를 보고 "아 그것도 있었지"라고
추가하고 싶어지는 게 정상이지만, 그렇게 하면 `MINE_ONLY`/`THEIRS_ONLY` 칸이 무의미해진다.
추가하고 싶은 항목은 `matrix.md`의 메모로 남긴다.

## R1 — 상대 호출

### 상대가 Codex일 때

```bash
SKILL="$HOME/.claude/skills/parallax"
EFFORT=<R0에서 사용자가 고른 값>

timeout 600 codex exec \
  -s read-only \
  -c model_reasoning_effort="$EFFORT" \
  --output-schema "$SKILL/reference/schema/finding.schema.json" \
  -o "$RUN/findings-theirs.json" \
  - < "$RUN/brief.md" | tee "$RUN/run.log"

grep -m1 '^session id:' "$RUN/run.log" | awk '{print $3}' > "$RUN/session-id"
```

`codex exec`는 시작 시 stdout에 헤더를 찍는다. 여기서 세션 id를 얻는다.

```text
OpenAI Codex v0.149.1
--------
workdir: ...
model: gpt-5.6-sol
approval: never
sandbox: read-only
reasoning effort: high
session id: 01a03929-afcd-7ed2-8e8e-a20a55419adb
--------
```

`~/.codex/session_index.jsonl`에는 `exec` 세션이 기록되지 않는다 (실측 확인).
인덱스 스냅샷 방식으로는 세션 id를 얻을 수 없으므로 반드시 헤더를 파싱한다.
헤더에서 id를 못 얻으면 R3를 세션 이어받기 없이 진행하되, 브리프 전문을 다시 첨부한다.

- `-s read-only` — 상대는 읽기만 한다. 예외 없다.
- `-C <경로>` — 대상이 현재 디렉터리 밖이면 추가한다.
- 저장소 밖이면 `--skip-git-repo-check`를 붙인다.
- 세션 id를 파일로 남기는 이유: R3에서 `--last`를 쓰면 그 사이 사용자가 연 다른 세션을
  이어받을 수 있다.

### 상대가 Claude일 때

```bash
timeout 600 claude -p --output-format json \
  --permission-mode plan \
  < "$RUN/brief.md" > "$RUN/findings-theirs.raw.json"
```

`--output-schema`에 해당하는 옵션이 없으므로, 브리프에 스키마 전문을 붙이고
응답에서 JSON 블록을 추출해 검증한다. `--permission-mode plan`으로 쓰기를 막는다.

### 실패 처리

| 상황 | 대응 |
| --- | --- |
| CLI 없음 / 인증 실패 | 재시도 없이 중단, 미검증 보고 |
| 타임아웃 (600초) | 1회 재시도. 재실패 시 중단 |
| 종료코드 ≠ 0 | 표준오류를 리포트에 첨부하고 1회 재시도 |
| JSON 파싱 실패 | 1회 재시도 (스키마 준수를 다시 강조). 재실패 시 중단 |
| findings 비어 있고 not_reviewed도 비어 있음 | 정상. "결함 없음" 관측으로 취급 |
| summary가 대상을 잘못 이해함 | **결과 폐기.** 브리프의 대상 지정을 고쳐 재실행 |

마지막 줄이 중요하다. `summary` 필드는 상대가 대상을 제대로 열어봤는지 확인하는 장치다.
엉뚱한 걸 검토한 findings를 매트릭스에 올리면 전부 `THEIRS_ONLY`로 잡혀 R3를 낭비한다.

## R2 — 매칭 판정

두 findings가 **같은 결함**인지 판정하는 기준. 하나라도 만족하면 같은 것으로 본다.

1. `location`이 같은 파일의 ±5줄 이내이고, `claim`이 같은 증상을 가리킨다
2. 문서 대상이면 같은 섹션이고 `claim`이 같은 문제를 가리킨다
3. `location`이 달라도 **같은 원인의 다른 발현**이다 (예: 같은 함수의 두 호출부)

3번은 신중하게 쓴다. 애매하면 **분리해서 양쪽 `*_ONLY`에 둔다.**
억지로 묶으면 실제 이견이 `AGREED`로 위장되고, 이 스킬의 유일한 가치가 사라진다.

`AGREED`인데 심각도가 다르면 **높은 쪽을 채택**하고 매트릭스에 차이를 기록한다.

### matrix.md 형식

```markdown
# 대조 매트릭스

프로파일: career | 모드: standard | 상대: Codex gpt-5.6-sol (effort=high)

## AGREED (3)
| id | 심각도 | 위치 | 요약 |

## MINE_ONLY (2)
| id | 심각도 | 위치 | 요약 |

## THEIRS_ONLY (4)
| id | 심각도 | 위치 | 요약 |

## 측정 불가 영역
- <양쪽 not_reviewed 합집합>
```

## R3 — 반박

### 상대에게 보낼 프롬프트

```markdown
앞서 당신이 검토한 대상에 대해, 다른 검토자가 아래 항목을 지적했습니다.
당신은 이 항목들을 지적하지 않았습니다.

각 항목을 다시 확인하고 판정해주세요.

- AGREE: 맞는 지적이다. 내가 놓쳤다.
- REFUTE: 틀린 지적이다. **반례를 제시하라** — 왜 문제가 되지 않는지 구체적으로.
- NEED_EVIDENCE: 판단할 근거가 부족하다. 무엇이 더 필요한지 적어라.

동의 압박을 느끼지 마세요. 틀린 지적은 틀렸다고 하는 것이 이 절차의 목적입니다.
반대로, 놓친 것을 인정하는 것도 실패가 아닙니다.

## 항목
<MINE_ONLY 항목들: id, 위치, 주장, 증거>
```

Codex라면 세션을 이어받아 보낸다. 대상을 다시 설명할 필요가 없다.

```bash
timeout 600 codex exec resume "$(cat "$RUN/session-id")" \
  -s read-only \
  -c model_reasoning_effort="$EFFORT" \
  -o "$RUN/adjudication-theirs.json" \
  - < "$RUN/rebuttal-prompt.md"
```

### 내 쪽 판정

`THEIRS_ONLY` 항목을 같은 기준으로 판정한다. 같은 규칙이 나에게도 적용된다 —
**상대가 지적했다는 이유만으로 동의하지 않는다.** 반박하려면 반례를 제시해야 한다.

### 종결

| 내 판정 | 상대 판정 | 결과 |
| --- | --- | --- |
| — | AGREE | `CONFIRMED` |
| AGREE | — | `CONFIRMED` |
| REFUTE (반례 있음) | — | `REJECTED` |
| — | REFUTE (반례 있음) | `REJECTED` |
| REFUTE (반례 없음) | — | `CONFIRMED` (반례 없는 반박은 반박이 아니다) |
| NEED_EVIDENCE 양쪽 | | `DISPUTED` |
| 엇갈림 | | `DISPUTED` |

`DISPUTED`가 나오면 **추가 라운드를 돌리지 않는다.** 양쪽 논거를 나란히 놓고 사용자에게 넘긴다.
2라운드, 3라운드를 돌리면 대개 근거가 아니라 인내심으로 결판이 나고, 그건 검증이 아니다.

## R4 — 재검증 (deep 모드)

사용자가 `CONFIRMED` 항목을 수정한 뒤 실행한다.

```markdown
아래 findings에 대한 수정이 적용되었습니다. 수정된 파일을 다시 읽고 판정해주세요.

- RESOLVED: 해소됐다
- NOT_RESOLVED: 여전히 문제다. 왜인지 적어라
- NEW_ISSUE: 이 수정이 새 결함을 만들었다

<CONFIRMED 항목 목록 + 수정된 파일 경로>
```

## 최종 리포트 템플릿

```markdown
# Parallax 교차검증 리포트

- 대상: <경로>
- 프로파일: <이름> (감지 신호: <신호>)
- 모드: <quick|standard|deep>
- 상대: <CLI> / <모델> / effort=<값>
- 실행: <run-id>

## 대조 매트릭스
| | 상대 지적 | 상대 미지적 |
| --- | --- | --- |
| 내가 지적 | AGREED n | MINE_ONLY n |
| 내가 미지적 | THEIRS_ONLY n | 측정 불가 |

## CONFIRMED (n)
<심각도 순. 각각 위치 · 주장 · 증거 · 누가 먼저 찾았는지>

## DISPUTED (n)
<항목별로 양쪽 논거를 나란히. 사용자 판단 필요>

## REJECTED (n)
<한 줄 요약 + 기각 근거>

## 측정 불가 영역
<양쪽 not_reviewed 합집합. 이 검증이 커버하지 못한 범위>

## 미검증
<실행하지 못한 단계, 사유, 남은 위험>
```

`CONFIRMED`가 0건이면 그대로 0건이라고 쓴다. 억지로 채우지 않는다.
두 모델이 독립적으로 봤는데 아무것도 안 나온 것도 정보다.
