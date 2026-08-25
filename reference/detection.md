# 경로 기반 프로파일 감지

현재 작업 경로에서 도메인을 읽어 검증 렌즈를 고른다.

프로파일을 잘못 잡으면 findings가 통째로 헛돈다. 코드 리뷰 렌즈로 자기소개서를 보면
"변수명이…" 같은 소리가 나오고, 문서 렌즈로 코드를 보면 경계 조건을 놓친다.

## 우선순위

| 순위 | 근거 | 비고 |
| --- | --- | --- |
| 1 | `--profile <이름>` 인자 | 사용자 명시. 무조건 우선 |
| 2 | `.parallax` 파일 | 프로젝트 고정값. 첫 줄에 프로파일 이름만 |
| 3 | 파일 시그니처 점수 | 주력 판정 |
| 4 | 디렉터리명 힌트 | 3번이 무득점일 때만 |
| 5 | `generic` | 폴백. 사용자에게 목적을 물어보고 진행 |

검증 대상이 특정 파일로 지정된 경우, **그 파일이 있는 디렉터리**를 기준으로 감지한다.
저장소 루트가 아니다 — 코드 저장소 안의 `docs/`는 `design-doc`이 맞다.

## 3순위 — 파일 시그니처 점수

대상 디렉터리와 그 상위 2단계까지 훑는다. 점수를 합산해 최고점을 채택한다.

### career (+5)

```bash
find . -maxdepth 2 \( \
  -iname '*이력서*' -o -iname '*자기소개서*' -o -iname '*자소서*' -o \
  -iname '*포트폴리오*' -o -iname '*지원서*' -o -iname '*공고*' -o \
  -iname '*resume*' -o -iname '*cv.*' -o -iname '*cover?letter*' -o \
  -iname '*portfolio*' -o -iname '*job?description*' \)
```

career 신호는 **가중치를 높게 준다.** 이력서 저장소에도 `.md`는 널려 있으므로
`design-doc`/`writing`과 붙으면 career가 이겨야 한다.

### code (+3)

```bash
ls package.json pyproject.toml setup.py Cargo.toml go.mod \
   build.gradle build.gradle.kts pom.xml composer.json Gemfile 2>/dev/null
```

추가 +1: 소스 확장자(`.py .ts .js .java .go .rs .rb .php .kt .swift .c .cpp`)가
전체 파일의 30% 이상.

### frontend (code 점수 + 2, code보다 우선)

```bash
ls tailwind.config.* next.config.* vite.config.* svelte.config.* \
   nuxt.config.* astro.config.* 2>/dev/null
ls -d components/ src/components/ app/ src/app/ styles/ src/styles/ 2>/dev/null
```

추가 +1: `.tsx .jsx .vue .svelte` 파일이 존재.

`code`와 `frontend`가 함께 잡히면 **`frontend`를 쓴다.** 렌즈가 더 좁고,
frontend 프로파일은 code의 카테고리를 잃지 않는다 (필요하면 브리프에 code 카테고리를 함께 넣는다).

### data (+4)

```bash
ls *.ipynb notebooks/ 2>/dev/null
ls -d data/ datasets/ 2>/dev/null
find . -maxdepth 2 \( -name '*.csv' -o -name '*.parquet' -o -name '*.xlsx' \) | head -1
grep -lE '^(pandas|numpy|polars|scikit-learn)' requirements.txt pyproject.toml 2>/dev/null
```

`.ipynb`가 있으면 `code` 점수를 무시하고 `data`를 쓴다. 노트북의 결함은 코드 결함과 종류가 다르다.

### design-doc (+3)

```bash
ls -d docs/ adr/ rfc/ design/ proposals/ 2>/dev/null
find . -maxdepth 2 -iname 'adr-*' -o -iname 'rfc-*' -o -iname '*spec*.md' \
  -o -iname '*design*.md' -o -iname '*proposal*.md' -o -iname '*설계*' -o -iname '*기획*'
```

추가 조건: 같은 디렉터리에 소스 파일이 거의 없어야 한다.
코드 저장소 루트에 `docs/`가 있다고 저장소 전체가 `design-doc`이 되지는 않는다.

### writing (+2)

```bash
ls -d .obsidian/ 2>/dev/null
ls -d _posts/ content/ notes/ vault/ 2>/dev/null
```

추가 +1: `.md`가 전체 파일의 70% 이상이고 위 어느 신호도 안 잡힘.

`.obsidian/`이 있으면 강한 신호다 (+4).

## 4순위 — 디렉터리명 힌트

3순위가 무득점일 때만. 경로 문자열에 아래가 포함되면 해당 프로파일.

| 포함 문자열 | 프로파일 |
| --- | --- |
| `지원`, `채용`, `취업`, `job-`, `application` | `career` |
| `설계`, `기획`, `docs`, `spec`, `design` | `design-doc` |
| `분석`, `analysis`, `eda`, `notebook` | `data` |
| `블로그`, `blog`, `posts`, `글`, `vault` | `writing` |

## 감지 결과 표시

한 줄로 표시하고 그대로 진행한다. 매번 물으면 성가시고, 안 물으면 오탐이 통과한다.

```
프로파일: career (신호: 자기소개서_카카오_백엔드.md, 이력서.md, 공고.txt)
```

**예외 — 접전이면 묻는다.** 1위와 2위의 점수 차가 2점 이하이면 `AskUserQuestion`으로 확인한다.

```
프로파일이 애매합니다. 어느 렌즈로 볼까요?
  [1] design-doc  (신호: docs/, adr-007.md)
  [2] code        (신호: pyproject.toml, .py 41%)
```

## 프로젝트 고정값

같은 저장소에서 반복 사용한다면 루트에 `.parallax` 파일을 둔다.

```text
career
```

첫 줄의 프로파일 이름만 읽는다. 이후 감지 단계를 건너뛴다.

## 프로파일 조합

한 번의 검증에서 두 프로파일이 필요한 경우가 있다 (예: UI 컴포넌트 + 설계 문서를 함께 바꾼 PR).
이때는 **프로파일별로 따로 실행한다.** 카테고리를 섞으면 상대가 렌즈를 잃고 잡탕 findings를 내놓는다.
