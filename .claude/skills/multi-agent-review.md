---
name: multi-agent-review
description: |
  PR diff를 보안·성능·스타일 3개 전문 서브에이전트가 병렬로 리뷰하고
  결과를 우선순위 매트릭스로 취합해 PR 코멘트로 게시하는 패턴.
  새 프로젝트에서 다중 에이전트 코드 리뷰를 도입할 때 이 스킬을 참조한다.
---

# Multi-Agent Code Review — 패턴 청사진

새 프로젝트에서 코드 리뷰를 다중 전문 에이전트로 구성할 때 사용하는 재사용 패턴.
이 문서를 그대로 따라 하면 약 30분 안에 동일 시스템을 띄울 수 있다.

---

## 1. 에이전트 파일 기본 패턴

위치: `.claude/agents/<role>-reviewer.md`

```markdown
---
name: <role>-reviewer            # kebab-case. 오케스트레이터가 호출할 키
description: |                   # 언제·왜 부르는지를 구체적으로 작성
  이 에이전트는 …을 검토하는 전문가다. PR diff 가 …일 때, 또는
  …이 변경됐을 때 호출한다. 수정은 하지 않고 발견·근거·권장만 보고한다.
tools:                           # 최소 권한 원칙
  - Read
  - Glob
  - Grep
  - Bash                         # 필요한 에이전트만 (예: security)
model: sonnet | haiku            # 비용 정책 (아래 6절)
isolation: worktree              # 항상 worktree (2절)
---

# <Role> Reviewer

당신은 <역할> 전문 리뷰어다. ...
(검사 항목, 절차, 출력 형식, 규칙)
```

### frontmatter 5필드의 의미
| 필드 | 역할 | 작성 팁 |
|---|---|---|
| `name` | 식별자 | kebab-case, 역할 명사 + `-reviewer` |
| `description` | 호출 판단 근거 | "이 상황·이 파일 변경 시 호출" 처럼 구체적으로 |
| `tools` | 허용 도구 | 최소 권한. 코드 실행이 필요 없으면 `Bash` 제외 |
| `model` | 사용 모델 | 추론 필요 → `sonnet`, 패턴 매칭 → `haiku` |
| `isolation` | 격리 모드 | 항상 `worktree` (2절 참고) |

---

## 2. `isolation: worktree` — 설정 방법과 용도

각 서브에이전트가 **독립 Git Worktree**에서 실행되도록 한다. 그 결과:
- 한 에이전트의 파일 수정·체크아웃이 다른 에이전트에 영향 X
- 병렬 실행 시 git index 경쟁·충돌 없음
- 본인 작업 디렉터리만 보므로 컨텍스트가 더 작고 깨끗함
- 작업 끝나고 변경이 없으면 worktree는 자동 정리됨

설정은 frontmatter에 `isolation: worktree` 한 줄로 끝. Claude Code가 호출 시점에 `git worktree add` 를 알아서 처리한다.

**언제 worktree 없이 써도 되나?** 다음을 모두 만족할 때만:
- 에이전트가 read-only (Write/Edit 없음)
- 다른 에이전트와 절대 동시에 실행되지 않음
- 호스트 작업 디렉터리의 동적 상태(메인 Claude가 막 수정한 파일 등)를 같이 보고 싶음

리뷰 시스템에서는 위 조건을 만족하기 어려우므로 **항상 worktree** 를 권장.

---

## 3. 오케스트레이터 프롬프트 패턴 (병렬 호출)

### 로컬 인터랙티브 모드
```text
이 저장소의 변경 파일을 security-reviewer, performance-reviewer, style-reviewer
세 에이전트로 병렬 분석해줘.

변경 파일 확인: git diff main...HEAD --name-only

각 에이전트를 동시에 실행하고, 완료되면 결과를 심각도 순(높음 → 중간 → 낮음)으로
취합한 보고서를 /tmp/review/final.md 에 작성해줘.
```

### CI/CD headless 모드 (5절 참고)
```bash
claude -p "$PROMPT" \
  --permission-mode acceptEdits \
  --allowedTools "Read,Glob,Grep,Bash,Write,Task" \
  --output-format text
```

### 병렬 호출이 실제로 일어나려면
- 메인 Claude(오케스트레이터)에게 "**동시에**" / "**병렬로**" 라고 명시
- 메인 Claude가 한 번의 응답에서 `Task` 도구를 3번 호출해야 함 (단일 메시지의 multiple tool calls)
- 서브에이전트는 다른 서브에이전트를 호출할 수 **없다** — 반드시 오케스트레이터 경유

---

## 4. 결과 취합 — 심각도 순 정렬 & 중복 제거

### 결과 파일 저장 규칙 (필수 합의)
| 에이전트 | 저장 경로 |
|---|---|
| security-reviewer | `/tmp/review/security.md` |
| performance-reviewer | `/tmp/review/performance.md` |
| style-reviewer | `/tmp/review/style.md` |
| 오케스트레이터(최종) | `/tmp/review/final.md` |

### 취합 기준 3종 (반드시 적용)

**4-1. 중복 제거**
- 동일 `파일:라인` 을 지적하는 이슈가 여러 리뷰어에서 나오면 **최고 심각도 1건만** 유지
- 동률 시 우선순위: 보안 > 성능 > 스타일
- 제거된 이슈는 남긴 이슈의 "관련 지적" 항목에 한 줄 참조만 남김

**4-2. 우선순위 매트릭스 (3 × 3)**
- 가로: 영향도 (높음 / 중간 / 낮음)
- 세로: 수정 난이도 (쉬움 / 중간 / 어려움)
- 각 칸에 `[카테고리][심각도] 파일:라인` 형식의 짧은 식별자 나열
- 정렬: **쉬움×높음 → 중간×높음 → 쉬움×중간** 순 (ROI 높은 순)

**4-3. 높은 심각도 요약**
- `final.md` **최상단**에 `## 🔴 즉시 수정 필요 (높은 심각도)` 섹션 강제
- `[높음]` 이슈만 한 줄 요약 + 파일:라인 + 1줄 권장으로 추출
- 본문이 길어도 이 섹션만 봐도 다음 액션을 알 수 있게

### final.md 고정 구조
```
## 🔴 즉시 수정 필요 (높은 심각도)   ← 4-3 요약
## 📊 우선순위 매트릭스             ← 4-2 매트릭스
## 상세 결과                       ← 심각도 순(높음→중간→낮음), 4-1 적용 후
## 분석 메타                       ← 변경 파일 수, 각 리뷰어 실행 상태
```

---

## 5. GitHub Actions Headless 모드 연동

### 워크플로우 7대 핵심
```yaml
on:
  pull_request:
    types: [opened, synchronize]   # PR 생성·새 커밋만
    paths:
      - 'src/**/*.py'              # 코드 파일만 (문서·설정 제외)
      - 'src/**/*.ts'

permissions:
  contents: read                   # 최소 권한
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    timeout-minutes: 15            # 적정 상한 (병렬 3에이전트 + 취합)
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }   # 3-dot diff 위해 전체 히스토리

      - name: Run claude
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}   # 키는 Secrets
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          claude -p "$PROMPT" --allowedTools "Read,Glob,Grep,Bash,Write,Task"
```

### 비용 가드 (필수)
```yaml
- name: Skip oversized PR (>10 files)
  if: fromJSON(steps.changed.outputs.count) > 10
  run: |
    gh pr comment "${{ github.event.pull_request.number }}" \
      --body "⚠️ 변경량이 너무 많습니다. 10개 이하로 분리하는 것을 권장합니다."
```
이후 단계들도 `if: fromJSON(...) <= 10` 로 가드해서 분석을 생략한다.

### GitHub MCP 연결 (.claude/settings.json)
```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}" }
    }
  }
}
```
- CI에서 `secrets.GITHUB_TOKEN` 을 step env로 주입하면 `${GITHUB_TOKEN}` 치환으로 MCP가 PR API 호출 가능
- **시크릿은 파일에 박지 말 것** — 항상 환경변수 참조

### 결과 게시
```yaml
- name: Post review comment
  if: always()
  run: |
    if [ -s /tmp/review/final.md ]; then
      gh pr comment "${{ github.event.pull_request.number }}" --body-file /tmp/review/final.md
    fi
```

---

## 6. 에이전트별 모델 선택 기준 (비용 최적화)

| 에이전트 | 작업 성격 | 모델 | 이유 |
|---|---|---|---|
| security-reviewer | 데이터 흐름·인가 흐름 추론 | `sonnet` | 컨텍스트 연결 + 가정 추론 필요 |
| performance-reviewer | 코드 흐름·복잡도 분석 | `sonnet` | 호출 그래프·자료구조 판단 필요 |
| style-reviewer | 명명·길이·중복 패턴 매칭 | `haiku` | 단순 규칙 위반 검사 — 가벼운 모델로 충분 |

**효과**: 세 에이전트 모두 `sonnet`으로 두는 것 대비 약 **30% 비용 절감** (style 분량이 비슷할 때 기준).

### 모델 강등 판단 체크리스트
- 코드 변경 추적이나 다단 추론이 **불필요**한가?
- 규칙이 명시적이고 위반이 표면에 드러나는가?
- 오탐(false positive) 허용 폭이 큰가? (스타일은 보통 그렇다)
- 셋 다 YES → `haiku` 로 강등 검토

추론 깊이가 필요한 보안·성능은 강등하지 말 것. 잘못된 리뷰는 비용 절감보다 훨씬 비싸다.

---

## 7. 빠른 도입 체크리스트

새 프로젝트에 이 패턴을 적용할 때 하나씩 확인:

- [ ] `.claude/agents/{security,performance,style}-reviewer.md` 3개 작성 (frontmatter 5필드)
- [ ] 모든 에이전트에 `isolation: worktree`
- [ ] `tools` 는 최소 권한 (security만 Bash, 나머지는 Read/Glob/Grep)
- [ ] style만 `model: haiku`, 나머지 `sonnet`
- [ ] `CLAUDE.md` 에 워크플로우·결과 파일 규칙·취합 기준 3종 명시
- [ ] `.claude/settings.json` 에 GitHub MCP (env: `${GITHUB_TOKEN}`)
- [ ] `.github/workflows/claude-review.yml` 작성 (pull_request 트리거, 15분 타임아웃)
- [ ] PR 크기 가드 (`>10` 스킵 + 경고)
- [ ] GitHub Secrets 에 `ANTHROPIC_API_KEY` 등록
- [ ] 로컬에서 병렬 실행 테스트 1회 통과 후 PR로 검증
