# ultragoal 해부 — 말을 증거로 받지 않는 실행 하네스

- 작성일: 2026-08-28
- 분석 대상: `references/gajae-code` (`v0.13.2`) 의 `ultragoal` 스킬 + 런타임 + 증거/가드 모듈
- 성격: 연구 노트. GJC 파이프라인 3부작의 마지막이다. [`deep-interview 해부`](./2026-08-28-deep-interview-ambiguity-scoring-and-revealed-preference.md)가 **명료성**, [`ralplan 해부`](./2026-08-28-ralplan-consensus-harness-and-reviewer-sandbox.md)가 **타당성**을 다뤘다면 이 문서는 **증명**을 다룬다. ultragoal은 파이프라인에서 **유일하게 실제로 코드를 바꾸는 스킬**이고, 그래서 게이트의 방향이 반대다.

## 0. 출처 표기

| 표기 | 뜻 |
| --- | --- |
| `[gjc]` | gajae-code 스냅샷에서 직접 확인 |
| `[도출]` | 위로부터 이 프로젝트를 위해 도출 |

---

## 1. 한 문장 요약

> **ultragoal은 "했다"는 모델의 자기 보고를 증거로 받지 않는 실행 하네스다. 표면 종류마다 다른 물리적 증거(스크린샷의 픽셀 분포, 터미널 제어코드, argv 재생 영수증)를 요구하고, 무엇을 요구할지는 모델의 판단이 아니라 변경된 파일 경로가 결정하며, 증명할 수 없으면 통과시키지 않는다. 그리고 실행 중에는 사용자에게 묻는 것 자체를 차단한다 — 앞의 두 스킬과 정반대다.**

---

## 2. 정체와 위치 `[gjc]`

### 2.1 게이트의 방향이 반대다

| | 묻는 것 | 시점 |
| --- | --- | --- |
| deep-interview | "뭘 원하는지 알고 있나" | 사전 |
| ralplan | "그렇게 만드는 게 맞나" | 사전 |
| **ultragoal** | **"한 게 맞나 — 증거를 가져와라"** | **사후** |

앞의 둘은 **허락(permission)**을 다루고 ultragoal은 **증명(proof)**을 다룬다.

### 2.2 분량 — 3부작 중 가장 극단적

| | `SKILL.md` | 프롬프트 fragment | 런타임 코드 | 배수 |
| --- | --- | --- | --- | --- |
| deep-interview | 1,046 | 126 | ~4,500 | 4.3배 |
| ralplan | 207 | 6 | ~2,850 | 14배 |
| **ultragoal** | **460** | 74 | **~8,700** | **19배** |

(배수는 `SKILL.md` 대비. 세 문서에서 동일 기준.)

런타임 내역: `ultragoal-runtime.ts` 5,408 · `ultragoal-guard.ts` 1,048 · `ultragoal-evidence.ts` 935 · `ultragoal-receipt-freshness.ts` 561 · `ultragoal-owner-loss-recovery.ts` 340 · `ultragoal-change-set.ts` 218 · `ultragoal-ask-guard.ts` 79 · `ultragoal-redteam-activation.ts` 75 · `commands/ultragoal.ts` 44

`[도출]` **되돌리기 어려운 일을 하는 스킬일수록 코드 비중이 커진다.** 계획이 틀리면 다시 짜면 되지만 코드를 잘못 만지면 남는다. 세 스킬의 배수(4.3 → 14 → 19)가 그 위험도 순서와 정확히 일치한다.

핵심 파일:

- [`ultragoal/SKILL.md`](../../references/gajae-code/packages/coding-agent/src/defaults/gjc/skills/ultragoal/SKILL.md)
- [`gjc-runtime/ultragoal-runtime.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ultragoal-runtime.ts)
- [`gjc-runtime/ultragoal-guard.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ultragoal-guard.ts)
- [`gjc-runtime/ultragoal-receipt-freshness.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ultragoal-receipt-freshness.ts)
- [`gjc-runtime/ultragoal-change-set.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ultragoal-change-set.ts)
- [`tools/ultragoal-ask-guard.ts`](../../references/gajae-code/packages/coding-agent/src/tools/ultragoal-ask-guard.ts)

### 2.3 상태 모델 — 원장이 정본이다

```
.gjc/_session-{id}/ultragoal/brief.md      원본 브리프
.gjc/_session-{id}/ultragoal/goals.json    목표 정체성과 상태의 정본
.gjc/_session-{id}/ultragoal/ledger.jsonl  체크포인트·영수증·blocker·스티어링·리뷰의 증명 스트림
```

SKILL.md의 규정:

> *"완료는 durable `goals.json`과 신선한 `ledger.jsonl` 영수증으로만 검증되며, **인라인 goal 상태로는 절대 검증되지 않는다.**"*

인라인 `goal` 툴은 **UX 브리지일 뿐**이고 CLI와 훅은 goal 상태를 절대 변경하지 않는다. 즉 대화 중 상태와 감사 가능한 상태를 분리하고, **후자만 신뢰한다.**

### 2.4 목표 생성과 병합 규칙

`@goal:` 구분자로 스토리를 나눠 `G001`, `G002`, … 로 만든다. 구분자 문법이 엄격하다 — 컬럼 0에서 시작하고 바로 뒤가 `:`/공백/줄끝이어야 하며, `@goalish`나 들여쓴 `@goal`은 일반 텍스트다. 제목도 본문도 없는 블록은 **거부**되고 자리표시자 목표를 쓰지 않는다.

그리고 **쪼개기 전에 합칠지부터 검사한다:**

> *"브리프를 얇은 스토리 여러 개로 쪼개기 전에, 후보 스토리들이 **validation-coupled**인지 확인하라. 그렇다면 하나의 goal로 병합하고, 그 안에서 executor 슬라이스를 펼쳐라."*

병합 판정 기준 4개:

- 같은 기능 스택 (한쪽 코드를 다른 쪽 없이 의미 있게 검증할 수 없음)
- 같은 수용 표면
- 같은 레드팀 표면
- 같은 최종 리뷰 경계 (묶어서만 승인 가능)

`[도출]` **이 규칙의 존재 자체가 상류 분해에 대한 자백이다.** deep-interview는 "독립적으로 성공/실패하는 결과물"로, ralplan은 "파일 변경과 의존성"으로 쪼갰는데, 실행 단계에서 **검증 축으로 다시 묶어야 한다**는 것. 세 단계의 분해 기준이 서로 다르다는 증거이며, 별도 설계 노트에서 다룬다 → [`분해와 수렴성`](./2026-08-28-decomposition-sufficiency-by-agent-convergence.md)

---

## 3. 축 ① 증거 강제 — 말을 증거로 받지 않는다 `[gjc]`

ultragoal의 성격을 규정하는 축이다.

### 3.1 표면마다 다른 물리적 증거

| 산출 표면 | 요구 증거 |
| --- | --- |
| GUI / 웹 | 유효한 자동화 트랜스크립트 **+ 균일하지 않은 스크린샷** |
| CLI | 안전한 argv 재생 (`schemaVersion: 1`, `kind: "cli-replay"`, `replaySafe: true`) 또는 감사된 `replayExempt` 경로 + 스크린샷/자동화/PTY 대체 |
| 네이티브 / 데스크톱 / TUI | 구조적으로 유효한 스크린샷, **터미널 제어코드가 포함된** PTY 캡처, 또는 앱 자동화 트랜스크립트 |
| API / 패키지 | 실제 아티팩트 파일 또는 `kind`에 `api`/`package`/`consumer`/`black-box`/`test-report`를 포함하는 타입화된 영수증 |
| 알고리즘 / 수학 | `kind`에 `property`/`boundary`/`edge`/`adversarial`/`failure`/`math`/`algorithm`/`test-report`를 포함하는 아티팩트 또는 영수증 |

그리고 못을 박는다:

> *"맨 `inlineEvidence` 텍스트만으로는 **어떤 표면도** 증명하지 못한다."*

`[도출]` **"테스트 돌렸고 통과했습니다"라는 문장은 증거가 아니다.** 스크린샷은 픽셀이 균일하지 않아야 하고(=진짜 렌더링), PTY 캡처에는 제어코드가 있어야 한다(=진짜 터미널). **위조하려면 실제로 그 일을 해야 하는 형태의 증거**를 고른 것이다.

### 3.2 자기가 쓴 테스트는 실행되지 않는다

> *"게이트는 **모델이 작성한 테스트 파일을 절대 실행하지 않는다.**"*

실행 가능한 것은 고정된 Bun 런타임의 `bun --version`과 리터럴 `bun -e "console.log(...)"` 뿐이다. 셸, 코드 문자열을 받는 인터프리터, 경로 지정 실행파일, 패키지/git/네트워크 변경 명령, `bun test`, 임의 argv는 전부 거부된다.

`[도출]` **에이전트가 자기 테스트를 짜서 자기가 통과시키는 것은 증명이 아니다.** 이 한 줄이 자기채점 문제의 실행 단계 버전에 대한 답이다 — 자매 문서들이 "모델의 자기 점수를 믿지 마라"였다면, 여기서는 "모델의 자기 테스트를 실행하지 마라"가 된다.

### 3.3 적용 여부를 판단이 아니라 경로가 결정한다

레드팀 스위트를 언제 요구할지를 런타임이 변경 파일 경로만 보고 정한다:

> *"**런타임이 change set으로부터 적용 여부를 결정하고, fail-closed다.** '이 변경이 실제로 뭘 배포하는지'에 대한 판단은 그것을 뒤집지 못하므로, 스킵 가능하다고 넘겨짚기 전에 경로를 확인하라."*

그리고 특정 레지스트리 파일(`settings-schema.ts`, `tools/index.ts`, `tools/renderers.ts`)은 **무조건** 트리거한다. 이유가 명시돼 있다:

> *"이 레지스트리들은 컴퓨터 제어 항목과 아닌 항목이 섞여 있어서, 경로만으로는 컴퓨터 제어 부분이 안 건드려졌음을 증명할 수 없다. 그래서 diff에 관련 내용이 하나도 없어도 요구한다."*

change set 캡처가 불완전했을 때도 요구한다.

`[도출]` **"증명할 수 없으면 요구한다"** — fail-closed의 교과서적 형태다. 그리고 문서가 그 불편함을 스스로 인정한다:

> *"컴퓨터 제어와 전혀 무관한 변경 — 예를 들어 설정 키 하나 추가 — 도 7개 필수 케이스에 걸린다. **통과하려고 그것들을 날조하지 말고, 게이트를 약화시키지도 마라.** 진짜 스위트를 내놓든가, blocker로 처리하고 운영자에게 에스컬레이션하라."*

권한 있는 우회는 별도 명령으로만 가능하다 — `gjc ultragoal record-critic-gate-override --evidence "<권한 근거>"`. **우회 자체가 기록되는 행위다.**

### 3.4 change set 계산의 세심함

[`ultragoal-change-set.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ultragoal-change-set.ts)의 `resolveGitBase`는 항상 `main`을 기준으로 삼지 않고 **가장 가까운 통합 베이스**를 찾는다. 주석의 이유:

> *"`dev`에서 딴 브랜치를 `main` 기준으로 잡으면 무관한 트렁크 이력이 쓸려 들어와 다른 사람의 변경이 이 스토리에 잘못 귀속되고, **변경 범위 기반 게이트를 잘못 발동시킨다.**"*

`[도출]` 게이트의 정확도가 **변경 집합 계산의 정확도에 종속**된다는 인식이다. 게이트를 촘촘히 만들수록 그 입력을 정확히 계산하는 일이 중요해진다.

### 3.5 레드팀 모드는 타입으로 활성화된다

executor에게 레드팀 프롬프트를 주입할지는 `executionMode: "ultragoal-red-team"` 타입 필드가 결정한다. 과제 텍스트 휴리스틱은 2순위이며, **`executorQa`를 그냥 언급한 것만으로는 활성화되지 않는다** (문서·필드명·부정문에서의 언급 때문). 알 수 없는 값은 fail-closed로 `undefined`가 되어 "켜짐을 발명하지 않는다".

---

## 4. 축 ② 동결 + 코호트 조인 `[gjc]`

무거운 게이트는 **경계 세대(boundary generation)당 한 번** 돈다 — 스토리마다도, 리뷰 패스마다도 아니다.

```
1. 경계의 누적 변경 집합에 대해 구현 검증을 실행
2. ★ 변경 집합을 동결한다. 리뷰 대상 소스에 대해 sourceHash 하나를 계산.
     이 세대의 모든 레인은 그 동일한 동결 스냅샷을 검사한다.
     다른 sourceHash를 들고 온 레인 판정은 거부된다.
3. 동결 스냅샷 위에서 코호트 레인 실행 — 세대당 cleaner 1 / architect 1 / qa 1.
     같은 세대의 두 번째 architect나 QA 레인은 거부된다.
4. architect 리뷰 (아키텍처면 / 제품면 / 코드면)
5. executor QA/레드팀 레인 — 깨뜨리려고 시도해야 하며 해피패스 확인이 아니다.
     승인된 계획/스펙/수용기준에서 출발해 사용자 대면 계약을 거쳐
     구현 코드는 마지막에 보조 증거로만 본다.
     계획/코드 불일치는 blocker이지 구현 의도로 덮을 항목이 아니다.
6. ★ 조인 후에 수리한다. 어떤 레인도 혼자 체크포인트할 수 없고,
     findings가 조인되기 전에는 수리 작업이 시작되지 않는다.
```

### 4.1 코드가 강제하는 두 가지

[`ultragoal-runtime.ts`](../../references/gajae-code/packages/coding-agent/src/gjc-runtime/ultragoal-runtime.ts) 2523~2556:

```
레인 해시 ≠ 코호트 해시
  → "every lane must inspect the same immutable source"

priorGenerationSourceHash == sourceHash
  → "a new generation requires a new frozen source"
```

**두 번째가 특히 중요하다.** 재검토하려면 **소스가 실제로 바뀌어야 한다.** 안 바뀐 코드를 다시 리뷰하는 것이 불가능하다.

`[도출]` ralplan의 레인 예산(§3.4, 횟수 기반)과 같은 목적을 **해시로** 달성한다. 횟수는 "몇 번 봤나"를 세지만 해시는 "볼 만한 게 바뀌었나"를 본다. **후자가 더 정확하고 튜닝 파라미터가 없다.**

### 4.2 "깨끗함"의 정의가 연언(conjunction)이다

체크포인트 `complete`가 허용되는 조건:

- `architectReview`의 `architectureStatus` / `productStatus` / `codeStatus`가 **전부** `CLEAR`
- `architectReview.recommendation`이 `APPROVE`
- executor QA 상태가 `passed`, iteration이 `passed`이고 `fullRerun: true`
- 코호트가 `joined: true`이며 **모든 레인이 깨끗하고 해시로 묶여 있음**
- 모든 증거 필드가 비어 있지 않음, 모든 필수 매트릭스 행이 존재, 모든 blockers 배열이 빔

`COMMENT`, `WATCH`, `REQUEST CHANGES`, `BLOCK`, 증거 누락, 얕은 매트릭스 행, 계획/코드 불일치, 비지 않은 blockers는 전부 비청결이다.

`[도출]` **부분 통과라는 개념이 없다.** 하나라도 빠지면 통과가 아니다. 실행 단계에서만 성립하는 엄격함인데, 이유는 명확하다 — 계획은 나중에 고칠 수 있지만 배포된 코드는 아니다.

### 4.3 ai-slop-cleaner — 내부 fragment

완료 게이트의 정리 청소는 `ai-slop-cleaner`라는 내부 sub-skill이 담당한다. deep-interview의 auto-research fragment와 같은 구조 — 특정 훅에서만 로드되고 슬래시 명령으로 발견되지 않으며 `skill://`로 해석되지 않는다.

- 활성 스토리의 변경 파일에 대한 **읽기 전용 탐지기+보고기**. 코드 편집·파일 쓰기·`.gjc/` 변경·체크포인트·goal 툴 호출·워크플로 spawn 모두 금지
- 전체 분류 체계로 blocking / advisory를 판정: fallback 위장, 중복, 죽은 코드, 불필요한 추상화, 경계 위반, UI/디자인 slop, 누락된 테스트
- **BLOCKING findings는 자체 수정 루프를 시작하지 않고 코호트 findings에 합류**한다. advisory는 게이트 보고서에만 남고 원장에 쓰이지 않는다
- 재귀 가드: 중첩된 `ralplan`/`team`/`deep-interview`/`ultragoal`을 spawn할 수 없고, 광범위하거나 아키텍처적인 findings는 리더에게 리뷰 blocker로 넘긴다

---

## 5. 축 ③ 일시정지 금지 — 앞의 둘과 정반대 `[gjc]`

### 5.1 `ask`가 차단된다

deep-interview와 ralplan은 `ask` 툴이 **핵심 게이트**다. deep-interview는 산문으로 질문 흉내내는 것까지 코드로 탐지해 막았다.

**ultragoal은 실행 중 `ask`를 차단한다.**

> *"`ask` 툴은 분류와 무관하게 활성 실행 중 계속 차단된다 — 미해결 결정은 사용자에게 프롬프트하는 대신 **durable blocker로 기록하라.**"*

[`ultragoal-ask-guard.ts`](../../references/gajae-code/packages/coding-agent/src/tools/ultragoal-ask-guard.ts)가 이를 도구 계층에서 강제하며, 차단 메시지가 대안을 함께 알려준다 (`gjc ultragoal record-review-blockers`).

세심한 부분: deep-interview와 ralplan은 "핵심 게이트가 `ask` 호출인 상류 계획 워크플로"로 취급되어 검사가 **현재 세션으로 스코프**된다. 다른 세션의 오래된 ultragoal 상태가 그 프롬프트를 납치하지 못하게 하기 위해서다.

### 5.2 blocker 분류와 정지 절차

> *"활성 Ultragoal 실행은 목표를 일시정지하고 사용자에게 물어봄으로써 blocker를 **포기해서는 안 된다.**"*

| 분류 | 뜻 | 처방 |
| --- | --- | --- |
| `resolvable` | 에이전트가 행동할 수 있는 모든 것 — 실패한 테스트, 미구현, 설치할 의존성, 애매하지만 추론 가능한 세부, 조사 | **절대 정지 금지.** 자율 해결을 소진하라: 조사 → `steer add_subgoal` → `executor` 위임 → 안 되면 blocker로 durable 보존하고 **다음 목표를 계속 스케줄** |
| `human_blocked` | 사용자만 행동할 수 있는 것 — 자격증명/시크릿, 수동·물리적 단계, 외부 승인/결정, 에이전트에게 없는 접근권 | 최후 수단이며 게이트가 걸린다 |

**애매하면 `resolvable`이 기본값이다.** 그리고 정지하려면 3단계를 밟아야 한다:

```sh
gjc ultragoal classify-blocker --classification human_blocked --evidence "<사람만 할 수 있는 구체적 의존>"
gjc ultragoal record-critic-verdict --terminus pause --classification-event-id <eventId> --verdict OKAY --evidence "<터미널 크리틱 근거>"
goal({"op":"pause"})
```

`--classification resolvable` 기록은 **감사 메모일 뿐 정지를 허가하지 않는다.**

`[도출]` **같은 행위(사람에게 묻기)의 가치가 단계에 따라 뒤집힌다.** 계획 단계에서는 미덕이고 실행 단계에서는 포기의 다른 이름이라는 판단이다. 우리 하네스에서도 단계마다 `ask`의 허용 여부를 **명시적으로 다르게** 정해야 한다는 뜻으로 읽힌다.

### 5.3 터미널 크리틱 게이트

실행 종료(완료 또는 정지) 지점에 fail-closed 리뷰가 한 번 있다. 읽기 전용 `critic` 역할이 `OKAY`를 줘야 하며, 스토리별로는 돌지 않는다.

비-`OKAY` 판정의 **run-level 상한이 5**다 — 모든 reopen 사이클을 가로질러 센다. 상한에 도달하면 정지와 최종 완료가 **모두 막히고**, 사람 또는 리더가 `record-critic-gate-override`를 기록해야 풀린다. **자동 정지 우회는 없다.**

---

## 6. `steer` — 되먹임의 완성된 형태 `[gjc]`

실행 중 발견이 스토리 분해를 바꿔야 함을 증명할 때 쓴다. **집계 목표와 제약은 고정된 채로.**

허용된 변형 6종:

```
add_subgoal              하위 목표 추가
split_subgoal            쪼개기
reorder_pending          순서 변경
revise_pending_wording   표현 수정
annotate_ledger          감사 메모만
mark_blocked_superseded  막힌 것을 대체됨으로 표시
```

불변조건:

- **자연어 요청은 추측하지 않고 거부**된다. "좀 바꿔줘"는 상태를 바꾸지 못한다
- `--evidence`와 `--rationale`이 **필수**
- **수락된 시도와 거부된 시도 모두** 원장에 구조화 감사 항목으로 append된다
- 집계 목표 / 원본 브리프 제약 / 품질 게이트 / 완료 상태는 편집 불가
- **하드 삭제 금지** — 대체된 목표는 스티어링 메타데이터와 함께 `goals.json`에 남고 스케줄링에서만 빠진다
- 대체물 없이 막힌 목표는 스케줄링에서 빠지되 **최종 완료를 계속 막는다**

### 6.1 예시의 rationale이 시사하는 것

```sh
gjc ultragoal steer --kind split_subgoal --goal-id G002 \
  --rationale "Splitting keeps each sub-goal independently verifiable."
```

`[도출]` **"왜 쪼개나 → 각 하위 목표가 독립적으로 검증 가능해지도록"** — 분해의 기준이 여기서 처음으로 명시된다. 다만 ultragoal은 그 기준을 **사람이 rationale에 적는 것**으로 두고 검증하지 않는다. 이 기준을 측정으로 바꾸는 것이 별도 설계 노트의 주제다.

### 6.2 되먹임에도 예산이 있다

`record-review-blockers`는 동일 목적(같은 trimmed objective + 같은 blocked goal + open 상태)의 blocker를 dedupe하고, 하나의 blocked goal당 **미해결 review_blocker 하강을 3단계로 제한**한다. 4번째 시도는 타입화된 `review_blocker_recursion_cap` 종료 handoff를 던진다 (CLI exit 1, 운영자 가시 마커). 명시적으로 — *"findings를 조용히 자동 완료 처리하지 않는다."*

`[도출]` **되먹임 경로를 여는 순간 순환 가능성이 생기므로, 여는 것과 예산을 정하는 것은 한 세트다.** 되먹임을 1급 상태로 만들려는 설계에 그대로 적용된다.

---

## 7. 발견 / 비판 `[도출]`

### 7.1 `ask` 금지는 양날이다

자율성은 얻지만 deep-interview가 세운 원칙과 정면으로 충돌한다 — *"인터뷰는 코드베이스가 아니라 인간과 하는 것이다."*

실행 중 **진짜 모호함**을 발견하면 `resolvable`로 분류해 추론하고 진행하거나 blocker로 쌓고 넘어간다. 사용자는 나중에야 안다. `human_blocked` 통로는 문턱이 높고(분류 → 크리틱 승인 → 정지), **기본값이 `resolvable`**이므로 실무에서는 추론 쪽으로 기운다.

**이는 트레이드오프이지 버그가 아니다.** 다만 실행 중 발견된 모호함이 조용히 가정으로 흡수될 위험이 있고, 그것은 deep-interview가 그토록 공들여 막으려던 바로 그것이다. **상류에서 잡은 것을 하류에서 다시 흘리는 구조**다.

### 7.2 ID 공간이 세 번째로 늘었다

자매 문서에서 지적한 문제가 여기서 확정된다:

```
deep-interview:  surface:review, artifact:audit-report   (SHA-256 봉인)
ralplan:         plan:cache-layer                        (계획 내부 ID)
ultragoal:       G001, G002, G003                        (스토리 ID)
```

`goals.json`은 `G001`이 상류의 어떤 잠긴 ID를 커버하는지 모르고 알 방법도 없다. 따라서 **"`surface:review`가 실제로 구현됐는가"를 자동으로 확인할 수 없다.**

**각 단계 안에서는 해시까지 써가며 엄격한데 단계 사이는 비어 있다.** 이것이 우리 프로젝트의 추적성 검증기가 채울 자리다.

### 7.3 증거 규칙이 표면별 하드코딩

"GUI면 스크린샷, CLI면 argv 재생, API면 아티팩트"가 코드에 박혀 있어 새 산출 표면이 생기면 규칙 추가가 필요하다. 확장성 문제이지만 **fail-closed라서 모르는 표면은 통과하지 못한다** — 안전한 쪽으로 틀린 설계다.

---

## 8. 3부작 종합 `[도출]`

| | deep-interview | ralplan | ultragoal |
| --- | --- | --- | --- |
| 묻는 것 | 뭘 원하나 | 어떻게 만드나 | **했다는 증거** |
| 게이트 성격 | 명료성 | 타당성 | **증명** |
| `SKILL.md` : 코드 | 1 : 4.3 | 1 : 14 | **1 : 19** |
| 척도 | 기수 (0~1 실수) | 서수 (3단계 토큰) | **이진 + 증거 유무** |
| `ask` 툴 | **핵심 게이트** (우회 탐지까지) | 핵심 게이트 | **차단됨** |
| 사람의 역할 | 매 라운드 답변 | 최종 승인 | **최후 수단** |
| 실패 시 거동 | 우회로로 빠져나감 | `PLANNING-STUCK` (실행 영구 차단) | blocker 기록 후 계속 진행 |
| 반복 제어 | 라운드 100 하드캡 | 반복 5 / 레인 1 (횟수) | **sourceHash 변경 요구** (해시) |
| 신뢰하지 않는 것 | 모델의 **자기 점수** | 리뷰어의 **일관성** | 모델의 **자기 보고 전체** |

`[도출]` **마지막 줄에 진행이 보인다.** 단계가 내려갈수록 모델을 덜 믿는다. deep-interview는 점수를 clamp하고, ralplan은 리뷰어 간 충돌을 강제 판정시키고, ultragoal은 **아예 "말"을 증거로 받지 않는다.** 위험도가 올라갈수록 신뢰를 거두는 것이 일관된 설계 원칙으로 보인다.

---

## 9. 이 프로젝트에 가져갈 것 `[도출]`

### 9.1 증거 설계 원칙 4개

1. **위조하려면 실제로 그 일을 해야 하는 증거를 고른다.** "통과했습니다"라는 문장이 아니라, 픽셀 분포·제어코드·실행 영수증처럼 **꾸미기 어려운 산출물**을 요구한다.
2. **모델이 만든 검증 도구를 실행하지 않는다.** 자기 테스트를 자기가 통과시키는 건 증명이 아니다. 우리 경우엔 *"에이전트가 작성한 커버리지 스크립트를 그대로 신뢰하지 않는다"*가 된다.
3. **적용 여부를 판단이 아니라 데이터가 결정한다.** "이건 해당 없어 보인다"는 모델의 판단으로 게이트를 스킵할 수 없어야 한다. 경로·타입·집합 연산 같은 관측값이 결정한다.
4. **증명할 수 없으면 요구한다.** 섞여 있어서 판별 불가능한 대상은 무조건 게이트를 건다. 불편함을 감수하는 쪽이 기본값이다.

### 9.2 그대로 훔칠 장치 3개

| 장치 | 우리 대응물 |
| --- | --- |
| **`sourceHash` 동결** | "모든 레인이 비트 단위로 동일한 입력을 봤다"를 주장이 아니라 **증명**으로 만든다. 별도 설계 노트의 수렴성 측정에 그대로 쓰인다 |
| **`steer` 5종 세트** | 타입 고정 · 자연어 거부 · evidence+rationale 필수 · 거부도 기록 · 하드 삭제 금지. **되먹임을 1급 상태로 만들 때의 최소 구성** |
| **재귀 상한 3** | 되먹임을 여는 순간 예산도 같이 정한다 |

### 9.3 단계별로 `ask` 정책을 명시하라

세 스킬의 `ask` 정책이 정반대라는 것이 우연이 아니다. 우리 하네스도 **단계마다 "사람에게 물어도 되는가"를 명시적으로 정해야** 한다. 기본값을 하나로 통일하는 것이 오히려 틀렸다.

---

## 10. 요약 표

| 질문 | 답 |
| --- | --- |
| ultragoal은 무엇인가 | 파이프라인에서 유일하게 코드를 바꾸는 실행 하네스. 게이트가 사전 허락이 아니라 **사후 증명** |
| 왜 하네스인가 | `SKILL.md` 460줄 대비 런타임 약 8,700줄 (19배). 3부작 중 최대 |
| 무엇을 신뢰하지 않나 | **모델의 자기 보고 전체.** 완료는 durable 상태와 신선한 영수증으로만 검증 |
| 증거를 어떻게 요구하나 | 표면별로 다른 **물리적 증거** — 균일하지 않은 스크린샷, 터미널 제어코드, argv 재생 영수증. 텍스트 서술은 어떤 표면도 증명 못 함 |
| 가장 강한 한 줄 | **"게이트는 모델이 작성한 테스트 파일을 절대 실행하지 않는다"** |
| 무엇을 요구할지는 누가 정하나 | 모델이 아니라 **변경된 파일 경로**. fail-closed이며 판단으로 뒤집을 수 없음 |
| 동결 메커니즘 | 경계 세대마다 `sourceHash` 하나. 레인 해시 불일치 거부, **새 세대는 새 해시 요구** |
| `ask`는 | **차단됨.** 실행 중 사용자에게 묻는 것은 포기의 다른 이름이라는 판단 |
| 정지하려면 | `human_blocked` 분류 → 터미널 크리틱 승인 → 그제서야 pause. 기본값은 `resolvable` |
| 되먹임 | `steer` 6종, 자연어 거부, evidence+rationale 필수, 거부도 기록, 재귀 상한 3 |
| 남은 구멍 | 상류 ID(`surface:review`)와의 매핑 부재. 각 단계 안은 엄격, **단계 사이는 비어 있음** |

---

## 11. 관련 문서

- [`2026-08-28-deep-interview-ambiguity-scoring-and-revealed-preference.md`](./2026-08-28-deep-interview-ambiguity-scoring-and-revealed-preference.md) — 명료성 게이트
- [`2026-08-28-ralplan-consensus-harness-and-reviewer-sandbox.md`](./2026-08-28-ralplan-consensus-harness-and-reviewer-sandbox.md) — 타당성 게이트
- [`2026-08-28-decomposition-sufficiency-by-agent-convergence.md`](./2026-08-28-decomposition-sufficiency-by-agent-convergence.md) — 세 노트를 가로로 자르는 설계 제안
- [`2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md`](./2026-08-19-agent-harness-what-to-build-and-how-to-verify-it.md) · [`2026-08-19-skill-vs-harness.md`](./2026-08-19-skill-vs-harness.md)
- [`docs/prd.md`](../prd.md) · [`references/FROZEN.md`](../../references/FROZEN.md)

## 12. 미해결 / 후속

- `team` 미분석. ultragoal의 대안 실행 경로(tmux 기반 병렬 워커)이며, ultragoal이 "리더 소유"를 유지하는 방식과 대비된다. 우선순위는 낮음 — 우리 설계에 tmux 병렬화가 필요하지 않다
- `ultragoal-owner-loss-recovery.ts` (340줄) 미분석. 소유자 세션이 사라졌을 때의 복구 경로로, 우리가 장기 실행 워크플로를 만들면 참고할 만함
- `ultragoal-evidence.ts` (935줄)의 증거 타입 판정 로직 상세 미분석. §9.1의 원칙을 구현할 때 참고
- 3부작 전체에서 확인된 **ID 공간 단절**을 우리 추적성 검증기로 어떻게 메울지는 [`docs/prd.md`](../prd.md) 갱신 사안
