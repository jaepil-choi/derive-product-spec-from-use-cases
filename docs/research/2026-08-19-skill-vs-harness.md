# 스킬이란 무엇이고, 하네스와의 경계는 어디인가

- 작성일: 2026-08-19
- 분석 대상: Anthropic / OpenAI 공식 문서, `references/deepseek-harness` (`dsh-v0.1.0-rc.7`), `references/gajae-code` (`v0.13.2`)
- 성격: 연구 노트. 자매 문서인 [`2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md`](./2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md)가 "하네스에 무엇을 넣는가"를 다뤘다면, 이 문서는 "그중 무엇이 스킬이 되는가"를 다룬다.

## 0. 출처 표기

| 표기 | 뜻 |
| --- | --- |
| `[docs]` | Anthropic / OpenAI 공식 문서에서 직접 확인 |
| `[dsh]` | deepseek-harness 스냅샷에서 직접 확인 |
| `[gjc]` | gajae-code 스냅샷에서 직접 확인 |
| `[도출]` | 위로부터 이 프로젝트를 위해 도출 |

## 1. 한 문장 답

> **스킬은 "조건부로 로드되는 컨텍스트"이고, 하네스는 "모델 바깥에서 실행되는 코드"다. 스킬의 가치는 레시피의 내용이 아니라 *언제 로드될지를 스킬이 스스로 선언한다*는 것과, *레시피가 가리키는 결정론적 실행체를 함께 배달한다*는 것에서 나온다.**

"script가 없으면 skill이 의미 있나"라는 문제의식은 절반 맞다. 정확히는 **스킬은 세 가지 중 최소 하나를 배달해야 한다** — 모델이 모르는 **지식**, 모델이 하면 안 되는 **결정론적 실행**, 또는 모델이 어겨선 안 되는 **정책**. 이 셋 중 아무것도 없으면 그 스킬은 "모델이 이미 아는 것을 비싸게 다시 말하는 것"이고, 실제로 순수 손해다. Anthropic 문서 자체가 이 기준을 못박아 둔다: *"Default assumption: Claude is already very smart. Only add context Claude doesn't already have."* `[docs]`

## 2. 공식 정의 — 스킬은 무엇인가

### 2.1 구조와 3단계 로딩 `[docs]`

스킬은 `SKILL.md` 하나가 필수인 **디렉터리**다. 선택적으로 `scripts/`(실행), `references/`(컨텍스트로 읽힘), `assets/`(산출물에 쓰임)를 번들한다. 핵심은 각각이 **다른 시점에 로드**된다는 것이다.

| 레벨 | 언제 로드 | 토큰 비용 | 내용 |
| --- | --- | --- | --- |
| 1 · 메타데이터 | 항상 (부팅 시 시스템 프롬프트에) | 스킬당 ~100토큰 | `name` + `description` |
| 2 · 지침 | 스킬이 트리거될 때 | 5k 미만 권장 | `SKILL.md` 본문 |
| 3+ · 리소스 | 필요할 때만 | 접근 전까지 0 | 번들 파일. **참조 파일은 읽으면 컨텍스트에 들어오고, 스크립트는 실행되어 출력만 들어온다** |

이 3단계가 "progressive disclosure"의 실체다. 그리고 여기서 첫 번째 비대칭이 나온다 — **참조 파일과 스크립트는 비용 구조가 근본적으로 다르다.**

- `references/api.md` 를 읽으면 **파일 전체가 토큰이 된다.** 아낀 것은 "안 읽었을 때"뿐이다.
- `scripts/validate.py` 를 실행하면 **코드는 절대 컨텍스트에 들어오지 않고 출력만 들어온다.** `[docs]`

> *"When Claude runs `validate_form.py`, the script's code never loads into the context window. Only its output consumes tokens, which makes scripts far more efficient than having Claude generate equivalent code on the fly."* `[docs]`

즉 **스크립트가 없는 스킬의 컨텍스트 절약은 "안 읽는 것"이 최대치이고, 스크립트가 있는 스킬은 "무한한 코드를 상수 토큰으로 압축"한다.** 사용자의 직관이 정확히 이 지점을 짚었다.

### 2.2 필수 필드와 트리거 계약 `[docs]`

- `name`: 64자 이내, 소문자·숫자·하이픈만. `anthropic` / `claude` 예약어 금지.
- `description`: 1024자 이내, **무엇을 하는지 + 언제 쓰는지 둘 다**. 3인칭으로 쓸 것 ("I can help you..."는 금지 — 시스템 프롬프트에 그대로 주입되기 때문).

`description`은 스킬에서 **유일하게 항상 비용을 치르는 필드**이자, 100개 이상의 스킬 중 이것을 고를지 말지를 결정하는 유일한 근거다. 나머지 본문이 아무리 좋아도 여기서 안 걸리면 존재하지 않는 것과 같다. `[docs]`

### 2.3 자유도(degrees of freedom) — 문서의 가장 중요한 규칙 `[docs]`

| 자유도 | 형태 | 언제 |
| --- | --- | --- |
| 높음 | 산문 지침 | 유효한 접근이 여럿, 맥락이 결정, 휴리스틱이 안내 |
| 중간 | 파라미터 있는 의사코드/스크립트 | 선호 패턴이 있고 변형이 허용됨 |
| 낮음 | 파라미터 없는 특정 스크립트 | **깨지기 쉽고, 일관성이 결정적이고, 순서가 고정** |

문서의 비유: *"Narrow bridge with cliffs on both sides — there's only one safe way forward"* 에는 정확한 명령을, *"Open field with no hazards"* 에는 방향만. 그리고 명시적으로: *"Prefer scripts for deterministic operations: write `validate_form.py` rather than asking Claude to generate validation code."* `[docs]`

**이것이 "백테스트" 예시의 공식 문서판 근거다.** 백테스트 엔진은 절벽 사이의 좁은 다리고, 전략 아이디어는 열린 들판이다. 스킬 하나가 두 지형을 다 담되, 각각에 맞는 자유도를 준다.

### 2.4 OpenAI도 같은 포맷을 쓴다 `[docs]`

SKILL.md 포맷은 Anthropic에서 나와 오픈 표준으로 공개됐고 OpenAI(Codex CLI), GitHub Copilot 등이 채택했다. OpenAI의 공식 `skill-creator` 스킬은 같은 규칙을 거의 그대로 반복한다 — `scripts/` `references/` `assets/` 3분할, "같은 코드를 반복해서 다시 쓰고 있거나 결정론적 신뢰성이 필요하면 스크립트로", 그리고 **한 가지 추가 규칙**:

> *"Do not include auxiliary documentation like README.md, installation guides, changelogs, or setup procedures. The skill should only contain the information needed for an AI agent to do the job at hand."* `[docs]`

**스킬은 사람을 위한 문서가 아니다.** 이 한 줄이 "스킬을 문서처럼 쓰는" 가장 흔한 실패를 막는다.

Codex 쪽 역할 분담도 명확하다 — `AGENTS.md`는 **항상 켜져 있는** 레포 지침(셋업 명령, 테스트 명령, 코딩 표준), 스킬은 **필요할 때 켜지는** 과제별 전문성. `[docs]`

## 3. 스킬의 가치는 어디에서 나오는가

공식 문서와 세 레포를 겹쳐 보면 가치의 원천은 넷이고, **그중 진짜 방어 가능한 것은 두 개**다.

| # | 가치 원천 | 스크립트 필요? | 모델이 좋아지면 |
| --- | --- | --- | --- |
| 1 | **조건부 로딩** — 스킬이 자기 트리거 조건을 스스로 선언 | 아니오 | **남는다** (컨텍스트는 항상 유한) |
| 2 | **결정론 배달** — 계산을 토큰이 아니라 실행으로 | 예 | **남는다** (확률적 생성 ≠ 결정론) |
| 3 | 희소 지식 — 사내 스키마, 프로젝트 규약, 조직 정책 | 아니오 | 남는다 (학습 데이터에 없음) |
| 4 | 절차 서술 — "이 순서로 해라" | 아니오 | **흡수된다** |

4번이 대부분의 스킬이 실제로 하는 일이고, 자매 문서 §2의 2번 통찰("workflow는 가장 먼저 흡수된다")과 정확히 같은 판정을 받는다.

### 3.1 그래서 "레시피만 있는 스킬"은 무가치한가 — 아니오, 조건부다 `[도출]`

무가치해지는 경우는 명확하다: **모델이 이미 아는 절차를 적어 놓은 것.** "PDF는 문서 형식입니다"류. 공식 문서의 나쁜 예시가 정확히 그것이다.

가치가 남는 경우도 명확하다:

- **모델이 알 수 없는 사실** — `.gjc/` 경로 규약, 이 레포의 stage 이름, 사내 BigQuery 테이블에서 테스트 계정을 거르는 규칙.
- **모델이 알 수 없는 판단 기준** — "이 조직에서 blocking과 non-blocking을 가르는 선".
- **트리거 자체가 산출물인 경우** — dsh의 `record-browser-gif`는 "GUI를 바꾸는 모든 PR은 GIF를 포함해야 한다"는 **정책의 존재를 알리는 것** 자체가 절반의 가치다. 절차는 몰라도 되지만 의무의 존재는 알아야 한다.

**판정 규칙 `[도출]`**: SKILL.md의 각 문단에 대해 — *"이 문단을 지우면, 신선한 모델이 이 문단의 내용을 재발명할 수 있는가?"* 예면 지운다. 아니오면 그것이 이 스킬의 실제 지분이다.

### 3.2 스크립트의 진짜 이득은 토큰이 아니다 `[도출]`

문서는 스크립트의 이점을 넷으로 든다 — 신뢰성, 토큰, 시간, 일관성 `[docs]`. 하지만 하네스 관점에서 결정적인 것은 넷째 항목의 다른 얼굴이다:

**스크립트는 검증 가능한 중간 산출물을 만들 수 있는 유일한 수단이다.** 문서가 "plan-validate-execute" 패턴으로 부르는 것: 모델이 계획을 구조화된 파일로 쓰고 → 스크립트가 그 계획을 검증하고 → 그다음에만 실행한다. `[docs]`

이것은 자매 문서 §5의 "자기 채점 금지"와 같은 원리다. 스크립트는 **채점자를 응시자 바깥에 두는 가장 싼 방법**이다.

## 4. 하네스와 스킬의 경계

### 4.1 결정적 테스트 `[도출]`

> **"모델이 이 문장을 무시하면, 무슨 일이 일어나는가?"**
>
> - 아무 일도 안 일어난다 → 그것은 **스킬**이다 (advisory)
> - 도구 호출이 거부되거나, 쓰기가 막히거나, 상태가 진전되지 않는다 → 그것은 **하네스**다 (enforcer)

자매 문서 §3.4의 `advisory` / `enforcer` 라벨링이 여기 그대로 적용된다. **스킬은 정의상 전부 advisory다.** 스킬은 모델의 컨텍스트에 들어가는 데이터이고, 컨텍스트에 있는 것은 무엇이든 무시될 수 있다.

이것이 왜 중요한가: SKILL.md에 "절대 `.gjc/`를 직접 편집하지 마라"라고 대문자로 써 놓는 것과, 도구 계층에서 그 경로에 대한 write를 거부하는 것은 **전혀 다른 종류의 보증**이다. gjc는 둘 다 한다 — 그리고 후자가 실제 보증이다. `[gjc]`

### 4.2 3층 모델 `[도출]`

```text
┌─ 하네스 (런타임 코드) ──────────────────────────────┐
│  권한 · 게이트 · 상태 소유 · 반복 한도 · provenance  │  ← 못 어긴다
│  "스킬을 발견하고 로드하는 것" 자체도 여기            │
├─ 스크립트 (스킬이 번들한 결정론적 실행체) ───────────┤
│  계산 · 검증 · 변환 · 열거                          │  ← 안 틀린다
├─ SKILL.md (컨텍스트) ──────────────────────────────┤
│  희소 지식 · 판단 기준 · 자유도 배분 · 트리거 선언    │  ← 어길 수 있다
└─────────────────────────────────────────────────────┘
```

같은 규율이 세 층으로 나뉜다. 백테스트 예시로 매핑하면:

| 층 | 백테스트 스킬의 몫 |
| --- | --- |
| 하네스 | 실행 예산, 결과 아티팩트의 소유권, "백테스트 없이 전략을 승인" 차단 |
| 스크립트 | 엔진 자체, 데이터 로더, 룩어헤드 바이어스 검사기, 성과 지표 계산 |
| SKILL.md | 인터페이스 사용법, 이 조직이 보는 지표, 실패한 백테스트를 어떻게 읽는지 |

**모델이 하는 일은 전략을 만드는 것뿐이다.** 사용자의 원래 직관이 그대로 이 표다.

### 4.3 뒤집힌 관점: 스킬 자체가 하네스의 capability다 `[dsh]`

dsh는 스킬을 "파일 몇 개"로 보지 않고 **하나의 capability seam**으로 만들었다. `packages/skill/` 아래가 정확히 자매 문서 §4.2의 3역할이다:

| 패키지 | 역할 | ctx key |
| --- | --- | --- |
| `skill/` | Service Definition — 프로바이더 등록과 조회만 정의 | `ctx.skills` |
| `skill-filesystem/` | Provider — 로컬 파일시스템에서 발견 | `ctx.skills`에 등록 |
| `skill-badge/` | Provider — 번들 스킬 기여 | `ctx.skills`에 등록 |
| `tool-skill/` | Consumer — 모델용 카탈로그와 로더 툴 발행 | `ctx.tools`에 등록 |

레지스트리는 *"does not know whether skills come from local files, embedded plugin data, HTTP, or another backend"* — 그래서 원격 스킬 스토어로 바꿔도 모델이 보는 계약은 그대로다. `[dsh]`

그리고 가장 배울 만한 한 조각: **invocation policy를 2비트로 쪼갠 것.**

| 정책 | 모델에게 보임 | 사용자에게 보임 |
| --- | --- | --- |
| `modelInvocable: true, userInvocable: true` | O | O |
| `true, false` | O | X |
| `false, true` | X | O |
| `false, false` | X | X |

*"one discovery result can serve model-facing tools, human-facing commands, and trusted internal callers without conflating their catalogs."* `[dsh]`

**함의 `[도출]`**: 무거운 워크플로 스킬은 `modelInvocable: false, userInvocable: true`가 옳은 기본값일 수 있다. 모델이 알아서 발동하면 안 되고, 사용자가 명시적으로 부를 때만 걸리는 것. gjc가 `hide` 플래그 하나로 같은 문제를 푼다 — *"loaded and accessible via `/skill:<name>`, but omitted from the rendered system prompt's skill listing. Use for skills the user opts into explicitly rather than ones the model should auto-discover."* `[gjc]`

## 4.5 결정론적 출력으로 어떻게 행동을 제어하는가

여기가 실전의 핵심이다. **스크립트의 출력은 그 자체로 아무것도 제어하지 않는다.** 스크립트는 *판정*을 만들고, 하네스가 그 판정을 도구 파이프라인의 특정 지점에 꽂았을 때만 제어가 된다. 스킬은 그 판정을 어떻게 읽을지만 알려준다.

dsh의 도구 파이프라인이 그 꽂을 지점 전부를 한 줄로 보여준다 `[dsh]`:

```text
ctx.tools.execute()
  → tools/pre-execute      allow / deny / ask       (재정렬 가능한 waterfall)
  → 단조 guards            deny only                 (allow 결과가 없음)
  → tools/execute          around-dispatch wrapper   (신호 교체 가능)
  → tools/post-execute     accept / block + additionalContexts
  → finalizeContent
  → tools/result           불변, 관찰만
```

아래 11가지가 두 레포에서 실제로 확인된 제어 기제다. 위에서 아래로 갈수록 약해진다.

### A. 사전 차단 — 도구 호출 자체를 거부

가장 강한 제어. dsh의 `PreToolDecision`은 셋뿐이고, **인자 재작성이 의도적으로 빠져 있다** — *"Input rewriting is excluded because arguments are already logged and presented."* 히스토리·감사·UI·실행이 반드시 일치해야 하기 때문이다. `[dsh]`

```ts
type PreToolDecision = { kind: 'allow' } | { kind: 'deny'; reason: string } | { kind: 'ask'; reason?: string }
type ToolGuard = (execution: Readonly<ToolExecution>) => string | undefined
```

그리고 그 뒤의 **단조 guard**가 이 설계의 핵심이다:

> *"Because guards have no allow result, listener ordering cannot turn a denial back into permission."* `[dsh]`

**거부는 단조롭다.** 플러그인 등록 순서를 잘못 짜서 게이트가 열리는 사고가 원천적으로 불가능하다.

gjc는 같은 일을 프록시로 한다 — `edit`/`write`/`ast_edit`/`bash` 네 개만 감싸서 `execute` 직전에 `assertWorkflowMutationAllowed`를 부르고 `ToolError`를 던진다. `[gjc]`

### B. 거부 메시지가 곧 제어 신호다 `[gjc]`

이게 가장 실용적인 교훈이다. **막기만 하면 에이전트는 우회로를 찾는다.** gjc의 차단 메시지는 전부 "막았다 + 대신 이걸 써라" 형태다:

```text
.gjc workflow state and artifacts are runtime-owned. Agent mutation tools cannot
edit `.gjc/**`; use the sanctioned `gjc` CLI instead.
Use: gjc ralplan --write --session-id <...> --run-id <...> --stage <...>
```

```text
Ralplan planning phase boundary: keep refining the consensus plan and persist plan
artifacts through `gjc ralplan --write` (stage scratch files under a temp dir if
needed). Product-code mutation tools and patch execution are blocked while ralplan
is active; mutate only after the plan is approved and execution begins.
```

**거부 메시지에 다음 합법 수단이 없으면 루프가 수렴하지 않는다.** 에이전트는 막힌 벽 앞에서 변형된 시도를 반복하고, 그건 곧 예산 소진이다.

### C. 사후 차단 — 결과 자체를 교정 피드백으로 치환

```ts
type PostToolDecision =
  | { kind: 'accept'; content?: ContentBlock[]; value?: never; additionalContexts?: UserMessage[] }
  | { kind: 'accept'; value: JsonValue; content?: never; additionalContexts?: UserMessage[] }
  | { kind: 'block'; feedback: ContentBlock[]; additionalContexts?: UserMessage[] }
```

`block`은 값을 제거하고 교정 피드백을 담은 `isError`로 바꾼다. 즉 **검증기의 출력이 도구 결과를 대체한다.** content와 value를 동시에 교체할 수 없고, *"Content replacement is presentation policy, not confidentiality policy"* — 값을 숨겨야 하면 block하거나 value를 교체해야 한다. `[dsh]`

실사례가 `timeout-policy`다: 데드라인이 이기면 제공자의 결과를 구조화된 `TOOL_TIMEOUT` 에러로 교체한다. 교체 판정을 결과의 모양이 아니라 **신호(`timeoutOf`)로 키잉**한 이유까지 문서화되어 있다. `[dsh]`

### D. 컨텍스트 주입 — 막지 않고 알려주기 (advisory)

`repeat-tool-reminder`는 같은 도구를 동일 인자로 연속 호출하는 것을 세다가 임계값(`[3, 5, 8]`)에서 조언을 주입한다. *"never vetoes or rewrites a call"* — 결정은 전적으로 모델에게 남는다. `[dsh]`

배울 디테일 셋:

- **거부된 호출도 카운트된다.** 탐지가 `tools/post-execute`에 있어서 pre-execute가 거부한 호출에도 돈다. *"a model hammering a denied call is exactly the loop worth breaking."*
- **추적하지 않는 호출은 체인에 투명하다.** `todo_write`를 exclude하면 `grep X → todo_write → grep X`가 여전히 연속 2회로 센다. *"bookkeeping tools interleaved into a loop must not launder it."*
- **주입은 append-only라 KV 캐시를 안 깬다.** 조언이 재사용 가능한 프리픽스 뒤에 붙는다.

그리고 한계가 정직하게 적혀 있다 — *"Advisory only — escalating to block at a high threshold is not implemented, though `PostToolDecision` already supports blocking."* 즉 **막을 수 있는데 일부러 안 막았다.**

### E. 상태 소유권 이전 — "다 했다"고 선언할 수 없게 만들기 `[gjc]`

가장 구조적인 제어다. gjc는 `.gjc/**`에 대한 모든 mutation 도구를 막고, 상태 진전을 `gjc <workflow> --write` CLI로만 가능하게 했다.

**함의**: 에이전트가 "리뷰 통과했습니다"라고 *말해서* 상태를 진전시킬 수 없다. 진전하려면 검증기를 통과해야 한다. 자매 문서 §3.1의 "자기 채점" 한계가 여기서 물리적으로 해소된다.

가드가 얼마나 진지한지는 우회로 처리에서 드러난다 `[gjc]`:

- `/dev/stdout`, `/dev/stderr`, `/dev/fd/<n>`은 디스크립터 별칭이므로 차단. `exec 1<>src/product.ts; printf x >/dev/stdout`이 실제 레포 파일에 도달하기 때문. **`/dev/null`만 예외.**
- `exec` 리다이렉션은 정적으로 해석 불가 → **무조건 fail-closed.**
- `>|`, `<>`, `>&` 같이 평범한 `>` 스캐너가 놓치는 쓰기 형태를 별도 정규식으로 잡음.
- 아카이브/sqlite 경로, 내부 URL 스킴까지 대상 추출에 포함.

### F. Fail-closed writer + 영수증 — 스크립트 출력이 "다음 단계의 입장권"이 되는 것 `[gjc]`

`gjc ralplan --write`가 실제로 하는 거부 목록:

| 조건 | 처분 |
| --- | --- |
| 같은 `(stage, stage_n)`에 **다른 내용** | 덮어쓰기 거부 + `"Use a new --stage_n to record another pass."` |
| 원장 sha256 ≠ 실제 아티팩트 sha256 | 중복 제거 거부 |
| `pending-approval.md`가 final 스테이지와 바이트 불일치 | 거부 |
| finding의 `sourceReceipt.stage ≠ sourceRole` | 거부 (provenance 위조) |
| `sourceReceipt.stageN ≠ plannerStageN` | 거부 (다른 pass의 증거 재사용) |
| path/sha256이 run의 `index.jsonl` 행으로 해석되지 않음 | 거부 |
| 미해결 충돌이 하나라도 남음 | 거부 |
| 알 수 없는 conflict를 참조하는 disposition | 거부 |

전부 `exit code 2` + 사람이 읽을 수 있는 메시지. **스크립트의 출력 형식 자체가 제어 API다.**

크래시 복구까지 설계되어 있다: 아티팩트는 썼는데 원장 행이 없는 갭에서, **동일 내용 재시도는 행을 복구하고 정상 dedup 영수증이 되지만, 다른 내용 재시도는 평범한 덮어쓰기 거부가 유지된다.** 재시도 안전하면서 위조는 불가능하다.

그리고 짝을 이루는 규칙 — **역할 에이전트는 본문을 반환하지 않고 영수증만 반환한다** (`session_id`, `run_id`, `path`, `sha256`, verdict 토큰). 부모가 받는 것은 "주장"이 아니라 **검증 가능한 핸들**이다. 컨텍스트 절약과 위변조 방지가 같은 조치로 해결된다.

### G. 예산과 한도 — 런타임이 센다

- gjc: 재리뷰 **최대 5회가 runtime-enforced**(SKILL.md에 그렇게 명시). 레인당 상한 상수는 코드에(`RALPLAN_MAX_REVIEW_PASSES_PER_LANE_LIMIT = 10`), 설정 가능 범위는 검증된 필드로. `[gjc]`
- 소진 시 `PLANNING-STUCK` 마커가 서고, 그것이 `auto_handoff.effectiveTarget`을 전부 `off`로 해소한다. **한도 소진이 실패가 아니라 명시적 종결 상태가 된다.** `[gjc]`
- dsh: 도구가 자기 `ToolDefinition.timeoutMs`를 선언하면 `timeout-policy`가 강제. 단 **협조적이지 하드 킬이 아니다** — *"Declaring `timeoutMs` therefore means 'cooperative with `exec.signal`'"*. 신호를 무시하는 도구는 안 멈추므로, 신호를 전달하는 도구만 선언해야 한다. 셸/파일 도구는 일부러 선언하지 않았다. `[dsh]`
- `timeoutMs`는 **모델에게 절대 전송되지 않는다** — `schemas()`가 name/description/parameters만 화이트리스트한다. 예산은 하네스의 것이지 모델과의 협상 대상이 아니다. `[dsh]`

### H. 승인 게이트 — 사람에게 넘기고, 실패하면 닫기 `[dsh]`

`ApprovalOutcome`은 닫힌 집합이고 fail-closed다. `allowed-once`만 통과하고 `rejected`/`cancelled`/`unavailable`은 전부 거부.

> *"A missing, non-owning, throwing, or non-conforming answerer becomes `unavailable` rather than opening the gate."*

**답변자가 없거나, 던지거나, 규격을 안 지키면 문이 안 열린다.** 승인 채널의 장애가 곧 통과가 되는 흔한 버그를 타입으로 막았다.

### I. 턴 종료 제어 `[dsh]`

`ToolExecutionResult`에 `concludesTurn?: true`가 있다 — *"The agent loop stops after committing this successful result batch."* 그리고 중첩 실행에서는 *"only an authoritative nested success can conclude the enclosing run."*

**검증기가 통과하면 턴이 끝나고, 실패하면 계속 돈다.** 종료 조건을 모델의 판단이 아니라 결정론적 결과에 묶는 방법이다.

### J. 도구 가시성 자체를 좁히기

- dsh `ToolRestriction { allow?, deny? }`는 스코프가 **상속받는** 도구에만 적용된다. 여러 제약은 교집합이고, 스코프 자신의 등록은 면제 — 위임된 자식이 자기가 답하는 도구를 잃지 않도록. `[dsh]`
- gjc: 제한된 role agent는 **bash 환경 자체가 제한**되어 `--artifact` 파일 경로 수용이 비활성화된다. 그래서 `--artifact-env GJC_RALPLAN_ARTIFACT`로만 전달 가능하다. **규약을 문장으로 말하는 대신 환경을 좁혀서 강제한 것.** `[gjc]`
- 카탈로그에서 아예 빼기: dsh `modelInvocable: false`, gjc `hide: true`. `[dsh]` `[gjc]`

### K. 구조화 출력 강제 `[dsh]`

서브에이전트 `outputSchema`(object-rooted JSON Schema)는 in-process 백엔드에서 **forced capture tool**로 구현된다. 스키마가 미지원이면 start 자체가 실패한다.

산문 대신 검증된 객체만 나오게 만드는 것 — 상위 로직이 판정을 파싱할 수 있으려면 필수다. gjc의 `--lane-verdict <CLEAR|WATCH|BLOCK>` 같은 **열거형 토큰**이 같은 일을 CLI 레벨에서 한다.

### L. soft를 soft라고 못박기 `[dsh]`

dsh의 plan mode는 반례로서 교훈적이다:

> *"Plan mode is **soft guidance**. Sandbox mode and approval policy enforce restrictions independently; neither reads or writes plan state, so deployments configure them separately."*

계획 모드는 "제어처럼 보이지만" 실제 제어는 sandbox와 approval에 있다. **두 개를 섞지 않았다.** 그리고 `exit_plan_mode`는 플랜 모드가 꺼져 있을 때도 등록된 채로 남는다 — 진입/이탈이 도구 카탈로그를 바꾸지 않으므로 KV 캐시가 안 깨진다. 바뀌는 것은 프롬프트 섹션 하나뿐이다.

### 4.5.1 강도 순 정리

| 기제 | 꽂는 지점 | 강도 | 대표 |
| --- | --- | --- | --- |
| A 사전 차단 + 단조 guard | `tools/pre-execute` | **강제** | dsh `ToolGuard`, gjc mutation guard |
| E 상태 소유권 이전 | 도구 계층 + 단일 writer | **강제** | gjc `.gjc/**` 차단 |
| F fail-closed writer | 스크립트 exit code | **강제** | `gjc ralplan --write` |
| H 승인 게이트 | `ask` → approval service | **강제** | dsh `ApprovalOutcome` |
| G 예산/한도 | 런타임 카운터 | **강제** | 재리뷰 5회, `timeoutMs` |
| J 도구 가시성 | 레지스트리/환경 | **강제** | `ToolRestriction`, 제한된 bash |
| C 사후 차단 | `tools/post-execute` block | **강제** | `TOOL_TIMEOUT` 치환 |
| K 구조화 출력 | 스키마 검증 | **강제** | `outputSchema`, verdict 토큰 |
| I 턴 종료 | `concludesTurn` | 강제 | — |
| B 거부 메시지의 지시문 | 에러 문자열 | 유도 | gjc 차단 메시지 |
| D 컨텍스트 주입 | `additionalContexts` | **조언** | repeat-tool-reminder |
| L 프롬프트 섹션 | system prompt | **조언** | plan mode |

### 4.5.2 도출되는 설계 규칙 `[도출]`

1. **스크립트는 판정을 만들고, 제어는 하네스가 한다.** 스킬은 판정을 읽는 법만 말한다. 이 셋을 한 곳에 섞으면 셋 다 약해진다.
2. **거부는 단조롭게.** allow를 돌려주는 게이트를 만들지 않는다. 순서 사고로 열리는 문은 문이 아니다.
3. **거부 메시지에 다음 합법 수단을 넣는다.** 없으면 에이전트가 변형 재시도로 예산을 태운다.
4. **상태 쓰기 경로를 하나로 좁힌다.** 단일 정본 writer가 있어야 검증기가 병목이 될 수 있다. 우회로가 하나라도 남으면 검증기는 장식이다.
5. **영수증만 반환한다.** 본문 반환 금지 — 컨텍스트 절약과 위변조 방지가 같은 조치다.
6. **재시도는 안전하게, 위조는 불가능하게.** 동일 내용 재시도는 idempotent, 다른 내용은 거부.
7. **한도 소진은 실패가 아니라 종결 상태다.** `STUCK`이 없으면 무한 루프거나 거짓 성공이다.
8. **advisory를 enforcer인 척하지 않는다.** SKILL.md의 대문자 MUST는 강제가 아니다. 강제하려면 A~K 중 하나에 꽂아라.
9. **예산은 모델에게 보내지 않는다.** 협상 대상이 되는 순간 예산이 아니다.

### 4.5.3 이 프로젝트에 꽂을 지점 `[도출]`

| 기제 | 이 프로젝트에서 |
| --- | --- |
| A 사전 차단 | 사용자 `APPROVED` 이전에는 spec 정본 파일에 대한 write/edit 거부 |
| B 지시문 | 거부 시 `Use: spec write --node <id> --evidence <path>` 형태로 대안 지정 |
| E 상태 소유권 | 그래프 매니페스트는 런타임 소유. 에이전트는 CLI를 통해서만 노드/엣지 추가 |
| F fail-closed writer | 고아 노드, 끊어진 추적성, sha256 불일치, 미해결 충돌 → exit 2 |
| G 한도 | Critic 재검토 반복 한도 + `STUCK` 종결 상태 (현 PRD에 없는 상태) |
| K 구조화 출력 | Critic 판정을 열거형 토큰으로. 산문 판정 금지 |
| C/D 조언 | STALE 전파 결과를 `additionalContexts`로 주입하되 막지는 않기 |

## 5. deepseek-harness에서 얻는 교훈

### 5.1 스킬은 "절차"가 아니라 "증거 정책"으로 쓰여 있다

`record-browser-gif`는 표면상 GIF 만드는 레시피지만, 실제 내용의 대부분은 **무엇이 증거로 인정되는가**다:

- 하나의 스토리보드는 하나의 격리된 실행에서만 나온다. 실패하면 프레임을 버리고 처음부터 다시. **여러 실행의 프레임을 이어붙이지 않는다.**
- 완료 판정은 **정확 일치**여야 한다. `body.textContent.includes(...)` 같은 부분 문자열 검사는 사용자 프롬프트의 에코도 통과시키므로 금지.
- PR 본문을 편집하기 직전에 live head를 다시 읽고, 기록된 커밋과 다르면 **중단하고 재녹화**한다. 편집 후에도 head가 그대로인지 다시 확인한다.
- 고정 지연(`sleep`)은 상태 도달의 증거가 아니다.

이건 자매 문서 §5의 "세상을 검증하라"가 스킬 형태로 나타난 것이다. **절차는 흡수되어도 이 판정 기준은 남는다.** `[dsh]`

### 5.2 스킬은 무엇을 *하지 말지*를 더 많이 말한다

`dsh-pre-push-checks`의 실질은 "테스트를 돌려라"가 아니라 **"반사적으로 전체 스위트를 돌리지 마라"**다.

> *"Do not manually repeat a passing check merely because commit or push follows. In particular, do not run typecheck immediately before pushing solely to duplicate the pre-push hook."* `[dsh]`

그리고 커버리지를 숨기는 우회로를 하나씩 명시적으로 막는다 — `--passWithNoTests` 금지, 임계값 낮추기 금지, 커버되지 않은 파일을 감추려고 `--coverage.include`를 좁히기 금지.

**교훈 `[도출]`**: 좋은 모델일수록 "무엇을 하라"는 알아서 한다. 남는 지분은 **모델이 자연스럽게 빠지는 함정의 목록**이다. 스킬 본문에서 "이렇게 하세요"를 지우고 "이건 증거가 아닙니다"를 남기면, 대개 분량은 절반이 되고 가치는 늘어난다.

### 5.3 스킬 자체가 provenance와 승격 파이프라인을 갖는다

`dsh-code-review` 스킬은 손으로 고치지 않는다. 주기적으로 도는 유지보수 도구가:

1. 최근 머지된 PR의 **사람 리뷰 피드백**을 커밋 앵커와 함께 수집하고,
2. 피드백 시점 패치와 최종 랜딩 패치를 비교해 **실제로 반영된 것만** 남기고,
3. 서로 다른 두 리뷰어 어댑터가 독립적으로 분류·리뷰하고 (블로킹 지적이 없을 때까지 루프),
4. 후보 `SKILL.md` + **promotion manifest**(소스 커밋, 스킬 blob 해시, 피드백 URL, 랜딩 증거 범위, 어댑터 판정, 게이트 결과)를 만들고,
5. 승격 시 **현재 스킬이 기록된 소스 blob과 일치하는지 검사**하고 불일치면 중단한다.

*"Do not defer to 'the reviewers approved'; the maintainer contract is that the operator makes the final decision."* `[dsh]`

**교훈 `[도출]`**: 이것은 자매 문서 §7.2의 "실험 → 테스트 승격 파이프라인"이 스킬에 적용된 것이다. **스킬의 각 규칙은 출처(어떤 사람의 리뷰 코멘트)와 채택 증거(어떤 PR이 실제로 반영했는지)를 갖는다.** 규칙이 왜 거기 있는지 아무도 모르는 스킬은 반드시 부풀고, 부푼 스킬은 무시된다. 그리고 "후보가 안 나오는 날이 정상"이라는 문장이 이 설계의 절제를 보여준다.

### 5.4 두 개의 얇은 파일이 스킬 하나를 이룬다

dsh 스킬은 `SKILL.md` 옆에 `agents/openai.yaml`을 둔다 — `display_name`, `short_description`, `default_prompt`. 즉 **모델용 계약(SKILL.md)과 사람용 표면(런처 카드)이 분리**되어 있다. 이 레포도 이미 같은 구조를 쓰고 있다(`agents/openai.yaml`). OpenAI 문서의 "README를 스킬에 넣지 마라"와 짝을 이룬다 — 사람용 정보는 스킬 본문이 아니라 별도 파일로. `[dsh]` `[docs]`

## 6. gajae-code에서 얻는 교훈

### 6.1 무거운 워크플로 스킬은 런타임 없이는 성립하지 않는다

`ralplan`은 207줄 SKILL.md지만, 그 문서가 요구하는 보증의 대부분은 **문서 바깥의 코드가 강제한다**:

| SKILL.md가 말하는 것 | 실제로 강제하는 곳 |
| --- | --- |
| "계획 단계에서 제품 코드를 고치지 마라" | `workflow-mutation-guard.ts` — `edit`/`write`/`ast_edit`/`bash` 차단 |
| "`.gjc/`를 직접 편집하지 마라" | 같은 가드 — 리다이렉션·`exec` 재바인딩·아카이브 경로까지 fail-closed |
| "최대 5회 재리뷰" | *runtime-enforced* (문서에 그렇게 명시되어 있다) |
| "역할 에이전트는 본문이 아니라 receipt만 반환" | 제한된 role agent는 `--artifact-env`로만 쓸 수 있게 환경 자체를 제한 |
| "충돌은 명시적 처분 없이 병합 불가" | writer가 fail-closed — 미해결 충돌이 있으면 쓰기 거부 |

가드 코드의 주석 하나가 이 설계의 진지함을 보여준다: `/dev/stdout`, `/dev/stderr`, `/dev/fd/<n>`은 디스크립터 별칭이므로 `exec 1<>src/product.ts; printf x >/dev/stdout`로 실제 레포 파일에 도달한다 — 그래서 `/dev/null`만 예외로 둔다. `[gjc]`

**교훈 `[도출]`**: 스킬에 "MUST"를 쓰고 싶어질 때마다, 그것은 **하네스 티켓이 하나 필요하다는 신호**다. 강제할 수 없는 MUST는 그냥 소음이고, 반복되면 스킬 전체의 신뢰도를 깎는다.

### 6.2 스킬 간 체이닝은 도구여야 한다

gjc는 `skill(name, args)` 도구를 둬서 스킬이 다른 스킬로 넘어가게 한다. 그 도구 프롬프트의 금지 조항이 흥미롭다:

> *"Do NOT use this tool to 'remind yourself' of a skill you're already running. The current SKILL.md is already in your context."*
> *"The chained skill's planning/execution-boundary rules still apply. Chaining does not grant execution approval."* `[gjc]`

그리고 **네이티브 워크플로 caller가 아직 terminal phase에 도달하지 않았으면 체인을 거부한다.** 스킬 전환이 상태 기계의 전이로 취급되고, 전이 조건이 런타임에 있다.

**교훈 `[도출]`**: 스킬이 여럿이 되는 순간 "누가 지금 활성인가"가 상태가 된다. 그 상태를 모델의 기억에 맡기면 자매 문서 §3.1의 첫 번째 구조적 한계에 바로 부딪힌다.

### 6.3 프론트매터는 확장 지점이다

Anthropic 표준은 `name` + `description`뿐이지만 gjc는 필요한 만큼 늘렸다 — `argument-hint`(슬래시 커맨드 UX), `level`, `globs`, `alwaysApply`, `hide`(모델 자동 발견에서 제외), 그리고 플러그인 서브스킬용 `phase`/`parent`. 테스트 픽스처에 `invalid-phase`, `duplicate-parent-phase`, `malicious-mixed-root` 같은 이름이 있는 것으로 보아 **로더가 프론트매터를 스키마로 검증하고 거부한다.** `[gjc]`

**교훈 `[도출]`**: 프론트매터에 필드를 하나 추가하는 것은 곧 **로더가 검증할 수 있는 계약을 하나 만드는 것**이다. 산문으로 쓴 규약보다 훨씬 싸게 강제된다. 다만 자매 문서 §3.3의 플러그인 판정 3문항을 통과해야 한다 — 실제로 값이 달라지는가, 구현이 둘 이상인가, 이걸 바꿔서 산출물을 믿을 수 없게 만들 수 있는가.

### 6.4 반면교사는 그대로 유효하다

자매 문서 §10에서 이미 지적한 대로 `ralplan`은 무겁다(207줄, 최소 3 subagent, 최대 5회 루프). `deep-interview`는 **1046줄**이다 — Anthropic 권장치(500줄)의 두 배가 넘는다. gjc가 이걸 감당하는 방법은 **거의 항상 발동하지 않는 것**이다: 앵커(파일 경로·이슈 번호·심볼)가 하나만 있어도 통과시킨다.

**교훈 `[도출]`**: 스킬이 500줄을 넘어가면 그것은 문서 문제가 아니라 **분해 신호**다. 셋 중 하나다 — (a) 결정론적 부분을 스크립트로 못 빼냈거나, (b) 강제해야 할 부분을 하네스로 못 옮겼거나, (c) 조건부로 읽어야 할 부분을 `references/`로 못 쪼갰거나.

## 7. 이 프로젝트에의 적용

### 7.1 현재 상태 판정

현재 [`SKILL.md`](../../SKILL.md)은 스캐폴딩 템플릿 그대로다(`[TODO: ...]`). `description`만 이미 완성도가 높다 — 무엇을 하는지와 언제 쓰는지가 다 있고, "stop for user approval"까지 들어 있다.

[`docs/prd.md`](../prd.md)의 무게 분포는 자매 문서 §9.1의 지적을 그대로 받는다 — §6·§7(단계와 역할)이 가장 두꺼운데 그 부분이 가장 먼저 흡수된다.

### 7.2 3층 배분안 `[도출]`

| 층 | 이 프로젝트에서 |
| --- | --- |
| **하네스** | `APPROVED`는 사용자만 부여 · 단일 정본 writer · 반복 한도와 `STUCK` 종결 · sha256 provenance · join gate · 충돌 강제 처분 |
| **스크립트** | 그래프 매니페스트 스키마 검증 · 고아 노드 탐지 · 추적성 전수 순회(UC↔요구사항↔컴포넌트) · STALE 전파 계산 · 링크 무결성 · 종결 상태 게이트 |
| **SKILL.md** | 좋은 use case와 나쁜 use case를 가르는 기준 · Critic이 무엇을 blocking으로 볼지 · 시나리오 시뮬레이션 판단 · 무거운 절차의 진입 여부 |

**이 프로젝트에서 "백테스터"에 해당하는 것은 추적성/전파 검증기다.** 그것이 있으면 모델은 설계에 집중하고 부기는 결정론적으로 처리된다. 그것이 없으면 이 스킬은 "PRD 잘 쓰는 법"이 되고, 그건 좋은 모델이 이미 한다.

### 7.3 즉시 적용할 규칙 `[도출]`

1. **스크립트를 먼저 쓴다.** SKILL.md 본문은 스크립트가 정해진 뒤에 그 사이를 메우는 것으로 쓴다. 순서를 뒤집으면 반드시 산문이 부푼다.
2. **`modelInvocable`을 기본으로 켜지 않는다.** 이 워크플로는 무겁다. 사용자가 명시적으로 부르는 것이 옳고, 그 판정 게이트는 LLM이 아니라 결정론적 앵커 검사여야 한다(자매 문서 §11 미해결 항목).
3. **500줄 상한을 하드 게이트로 둔다.** 넘으면 `references/`로 쪼개거나 스크립트로 내린다.
4. **각 규칙에 출처를 남긴다.** dsh의 promotion manifest 축소판 — 어떤 실패에서 이 규칙이 나왔는지 한 줄. 출처 없는 규칙은 다음 정리 때 삭제 후보.
5. **"MUST"를 쓸 때마다 하네스 티켓을 만든다.** 강제할 수 없으면 "MUST"를 쓰지 않는다.

## 8. 요약 표

| 질문 | 답 |
| --- | --- |
| 스킬은 무엇인가 | 조건부로 로드되는 컨텍스트 + 함께 배달되는 결정론적 실행체 |
| 가치는 어디서 오는가 | ① 트리거 자기선언(컨텍스트 예산) ② 스크립트(토큰 대신 실행) ③ 희소 지식. **절차 서술은 아니다** |
| 스크립트 없는 스킬은 | 희소 지식이나 판단 기준을 담을 때만 유효. 모델이 재발명할 수 있는 절차면 손해 |
| 하네스와의 경계 | "모델이 무시하면 무슨 일이 일어나는가". 아무 일도 없으면 스킬, 막히면 하네스 |
| 스킬은 강제할 수 있는가 | **없다.** 스킬은 정의상 전부 advisory |
| 무거운 워크플로 스킬은 | 런타임 게이트 없이는 성립하지 않는다. gjc가 증명 |
| 스킬은 어떻게 안 썩는가 | 각 규칙에 출처와 채택 증거. 승격 파이프라인. 삭제를 기본값으로 |

## 9. 관련 문서

- [`2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md`](./2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md) — 하네스에 무엇을 넣는가
- [`docs/prd.md`](../prd.md) — 확정된 제품 결정
- [`references/FROZEN.md`](../../references/FROZEN.md) — 분석 대상 스냅샷의 고정 커밋

## 10. 출처

- [Agent Skills 개요 (Anthropic)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Skill authoring best practices (Anthropic)](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Equipping agents for the real world with Agent Skills (Anthropic Engineering)](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [openai/skills — skill-creator SKILL.md](https://github.com/openai/skills/blob/main/skills/.system/skill-creator/SKILL.md)
- [Codex CLI Agent Skills 가이드](https://itecsonline.com/post/codex-cli-agent-skills-guide-install-usage-cross-platform-resources-2026)
- [AI Agent Skills Guide 2026](https://www.thepromptindex.com/how-to-use-ai-agent-skills-the-complete-guide.html)
