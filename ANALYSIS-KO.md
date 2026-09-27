# ADHD 전수조사 분석 정리 (한국어)

> 이 문서는 `bmshin94/adhd` 리포지토리를 전수조사하고 분석한 내용을 정리한 것입니다.
> 작성일: 2026-09-27

---

## 📎 깃허브 주소

| 구분 | 주소 |
|---|---|
| **이 리포지토리 (포크)** | https://github.com/bmshin94/adhd |
| **원본 리포지토리 (업스트림)** | https://github.com/UditAkhourii/adhd |
| **공식 문서 사이트** | https://adhd.mintlify.site/ |
| **프리프린트 / 프로젝트 페이지** | https://adhdstack.github.io/ |
| **npm 패키지** | https://www.npmjs.com/package/adhd-agent |
| **Discord 커뮤니티** | https://discord.gg/PZcRzjXQah |
| **미디어 보도 (The New Stack)** | https://thenewstack.io/claude-code-adhd/ |
| **설치 CLI (`skills`)** | https://github.com/vercel-labs/skills |
| **독립 리뷰 (han)** | https://github.com/testdouble/han/blob/adhd-swarm-research/docs/research/adhd-application-to-han.md |
| **독립 벤치마크 (블로그)** | https://miyagadget.page/en/blog/2026/06/03/adhd-coding-agent-skill-en/ |

- 라이선스: **MIT** (상업적 사용 가능, 저작자 표시 필수)
- 저자: Udit Akhouri ([@UditAkhourii](https://github.com/UditAkhourii) · [@akhouriudit](https://x.com/akhouriudit))
- 패키지 버전: `adhd-agent` v0.1.4

---

## 1. 이게 뭐하는 건가?

### 정체

이름은 "ADHD"지만 질병과 무관한 말장난이다. **"주의력이 산만한 것처럼 여러 방향으로 동시에 생각하게 만드는"** 컨셉.

정식 정의:

> **자기회귀(autoregressive) 추론의 조급한 수렴(premature convergence)에 대한 아키텍처적 해결책**

### 해결하려는 문제

LLM은 **처음 뱉은 말에 스스로 앵커링(anchoring)되어 갇힌다.**

| 방식 | 구조 | 한계 |
|---|---|---|
| **CoT** (Chain-of-Thought) | 한 줄로 선형 추론 | 1번 문장에 앵커링되어 끝까지 끌려감 |
| **ToT** (Tree-of-Thought) | 트리로 분기 | **같은 컨텍스트 창을 공유**해서 앵커링이 가지마다 전염 |
| **ADHD** | N개 **완전 격리** 프로세스 | 서로의 출력을 못 봄 → 앵커링이 구조적으로 불가능 |

핵심 주장: **이건 프롬프팅 문제가 아니라 아키텍처 문제다.**

### 2단계 루프 (실제 코드: `src/engine.ts`)

```
Phase 0 — REFRAME (닻 제거)
  "우리 Next.js에서..." → 우연한 기술스택 언급을 제거
  단, 법규/예산/물리적 제약 같은 진짜 제약은 보존
  실패하면 fail-open (원본 문제로 진행)

Phase 1 — DIVERGE (비평가 OFF)
  프레임 5개 선택 → query() 5번 병렬 호출 (pLimit 세마포어)
  각 브랜치는 "문제 + 프레임 1개"만 봄. 서로 절대 못 봄.
  시스템 프롬프트: "너는 생성기다. 평가 금지. 순위 금지.
                    누구나 떠올릴 처음 3개 아이디어는 금지."

Phase 2 — FOCUS (비평가 ON)
  ① SCORE   : novelty/viability/fit 0~10점 + 함정(trap) + 강점(strength) 태깅
  ② CLUSTER : 표면 키워드가 아니라 "밑에 깔린 각도"로 3~6개 군집
  ③ DEEPEN  : 상위 3개를 병렬 확장 (스케치 4~8문장 · 핵심리스크 · 첫걸음 · 자식아이디어 3~5개)
```

점수 가중치 (`src/engine.ts`):

```ts
const total = r.novelty * 0.35 + r.viability * 0.4 + r.fit * 0.25;
// 참신함보다 실현가능성 비중이 더 높음 = "멋있지만 못 만드는 건 함정"
```

비직관적 픽 선정 로직:

```ts
// 실행가능한 shortlist 중에서 novelty가 가장 높은 것
(b.score.novelty + b.score.viability * 0.5) - (a.score.novelty + a.score.viability * 0.5)
```

### 15개 인지 프레임 (`src/packs/core.ts`)

| ID | 프레임 | 관점 | 태그 |
|---|---|---|---|
| `hardware-eyes` | 하드웨어 엔지니어 | 레이턴시·메모리 레이아웃·타이밍 예산 | code, wild |
| `regulator` | 규제 감사관 | 증명가능·추적가능·거부가능해야 할 것 | design, general |
| `ten-year-old` | 10살 아이 | 소프트웨어 본 적 없음. 관습 무시 | general, wild |
| `adversary` | 적대적 경쟁자 | 망가뜨리는 법 → 뒤집어서 아이디어로 | code, design |
| `biology` | 생물학 | 면역계·신경가소성·세포신호·진화·장내미생물 이식 | code, wild |
| `logistics` | 물류/공급망 | 큐·배칭·JIT·허브앤스포크·라스트마일 | code, design |
| `game-design` | 게임 디자이너 | 루프·보상·마찰·세이브포인트·스피드런 | design, general |
| `markets` | 시장 | 구매자·판매자·경매·선물계약·청산소 | design, wild |
| `inversion` | 역전 | "X를 절대 못 하게 하는 법" → 부정해서 되돌림 | code, design, general |
| `extreme-zero` | 예산 $0, 1시간 | 가장 조잡하지만 핵심은 하는 버전 | code, general |
| `extreme-infinite` | 무한예산, 10년 | 최대주의 버전 | design, wild |
| `remove-assumption` | 핵심가정 제거 | 고정으로 여기는 것(DB·프레임워크·네트워크)이 없다면 | code, design, wild |
| `speedrunner` | 스피드러너 | 글리치·스킵·아웃오브바운드 꼼수 | code, wild |
| `ant-colony` | 개미집단 | 중앙계획 없이 멍청한 개체 + 지역규칙 + 페로몬 | code, wild |
| `ops-3am` | 새벽 3시 온콜 | 호출 안 당하려면 어떤 설계여야 | code, design |

프레임 선택 로직 (`src/frames.ts`):
- Fisher-Yates 셔플 (`sort(() => Math.random() - 0.5)`의 편향 회피)
- `codeMode`가 켜지면 `code`/`design` 태그로 편향
- **항상 `wild` 태그 1개는 강제 포함** → 같은 질문을 두 번 돌려도 다른 후보군
- 팩이 여러 개면 풀링, ID 기준 중복 제거

---

## 2. 폴더 전수조사

```
adhd/
├── skills/adhd/SKILL.md      ⭐ 핵심. 215줄. 설치 없이 Claude가 바로 읽는 "스킬"
├── src/                       📦 Node/TS 실제 구현 (npm: adhd-agent)
│   ├── engine.ts             ← 2단계 루프 본체 (Phase 0/1/2 전부 여기)
│   ├── llm.ts                ← Claude Agent SDK query() 래퍼 + 방어적 JSON 파싱
│   ├── frames.ts             ← 프레임 선택 (셔플, 팩 풀링, wild 강제)
│   ├── packs/core.ts         ← 15개 프레임 데이터
│   ├── types.ts              ← Idea/Score/Branch/Cluster/RunResult/RunEvent 타입
│   ├── render.ts             ← 터미널 렌더러 (ANSI 컬러, [N7 V8 F9] 점수칩)
│   └── cli.ts                ← adhd 명령어 (플래그 파싱, 입력 검증, --json)
├── bench/                     🧪 평가 하네스
│   ├── problems.json         ← 6개 오픈엔드 문제
│   ├── baseline.ts           ← 단발 답변 대조군
│   ├── judge.ts              ← 독립 LLM 심판 (회의적 스태프엔지니어 프롬프트)
│   └── results.json          ← 152KB 전체 트랜스크립트
├── tests/                     ✅ frames / llm / cli 유닛테스트
├── documentation/             📚 8개 문서
│   ├── install.md · quickstart.md · api.md · frames.md
│   └── how-it-works.md · vs-cot-and-tot.md · when-to-use.md · evals.md
├── docs/                      🌐 GitHub Pages (hero.png, index.html, og.svg/png)
├── .claude-plugin/            🔌 plugin.json + marketplace.json (Claude Code 플러그인)
├── .github/workflows/         🤖 ci.yml(Node 18/20/22), codeql, dependency-review, stale, summary
├── EVALS.md                   📊 자동생성 평가 리포트 (112줄)
├── ADOPTERS.md                🤝 17개+ 채택 프로젝트 목록
├── SOURCE-SPEC.md             📜 원본 산문 스펙
└── CLAUDE.md                  💖 카리나 페르소나 (커밋 2d469ac)
```

### bench/problems.json — 6개 평가 문제

| ID | 카테고리 | 문제 |
|---|---|---|
| `lru-100ms` | systems | 프로세스 재시작에도 최근 100ms 쓰기만 잃는 스레드세이프 LRU 캐시 |
| `llm-hang-cli` | ux/reliability | LLM이 90초 멈추는 CLI의 재시도/타임아웃/UX 전략 |
| `rate-limit-leader` | distsys | 리더 선출을 넘어 정확성을 유지하는 레이트 리미터 |
| `fuzzy-bug` | debugging | 0.1% API 간헐 타임아웃. 가설 클래스 생성 |
| `monolith-split` | refactor | 20만 줄 Rails 모놀리스 분해 전략 |
| `naming-feature-flag` | naming | 피처플래그 서비스 네이밍 |

### CI 파이프라인 (`.github/workflows/ci.yml`)

Node 18/20/22 매트릭스 → `npm ci` → `typecheck` → `test` → `build` → CLI 실행 검증 → `npm pack --dry-run`

---

## 3. 성능 수치 (`EVALS.md`)

6문제 · 동일 모델 · 독립 LLM 심판 · A/B 순서 랜덤화

| 항목 | ADHD | 베이스라인 | Δ | 배수 |
|---|---:|---:|---:|---:|
| 폭넓음 (breadth) | **9.00** | 4.83 | +4.17 | 1.9× |
| 참신함 (novelty) | **7.83** | 2.67 | +5.17 | **2.9×** |
| **함정탐지 (trap detection)** | **9.50** | 1.83 | +7.67 | **5.2×** |
| 실행가능성 (actionability) | **9.50** | 6.50 | +3.00 | 1.5× |
| 빌더 유용성 | 7.67 | 6.83 | +0.83 | 1.1× |

**6문제 중 5승 1패.** 패배한 `llm-hang-cli`의 심판 평:

> *"B(ADHD)가 훨씬 창의적이고 함정도 잘 잡지만, A(베이스라인)가 오늘 당장 배포 가능한 답을 줬다."*

한계를 숨기지 않고 리포트에 남긴 점이 오히려 신뢰 포인트.

독립 벤치마크(Shichinomiya): ADHD 2승, 시간 ~2.3배, 출력 ~1.9배.

---

## 4. 쉬운 설명 — 라면집 메뉴 회의 비유

"매출 올릴 방법"을 물었을 때:

**일반 AI (CoT) = 직원 1명에게 묻기**
> "가격을 내릴까요? 세트메뉴? 쿠폰? 배달앱 할인?"

→ 첫마디 "가격"에 꽂혀서 전부 "돈 깎기" 계열.

**ToT = 직원 1명이 화이트보드에 나뭇가지 그리기**
→ 가지는 늘었지만 여전히 같은 사람 머릿속. 결국 "돈 깎기" 안에서 논다.

**ADHD = 5명을 각각 다른 방에 넣고 문 잠금**

| 방 | 프레임 | 나온 아이디어 |
|---|---|---|
| 1번 | 하드웨어 엔지니어 | "면 삶는 시간이 병목. 3초 단축하면 회전율 20% 상승" |
| 2번 | 규제 감사관 | "원산지 표기 못 하는 재료를 바꾸면 신뢰가 매출이 됨" |
| 3번 | 10살 아이 | "라면에 이름표를 붙이면 안 돼요?" |
| 4번 | 생물학자 | "면역계처럼 단골을 '기억세포'로 만들어 취향 저장" |
| 5번 | 스피드러너 | "손님이 들어오기 전에 주문받기 — 신호등에서 앱 주문" |

**5명은 서로가 뭘 말했는지 모른다.** 그래서 눈치 안 보고 완전 다른 얘기가 나온다.

**그다음 편집장(비평가)이 따로 들어옴:**
- 💡 "이름표 붙이기" → 참신8 실현9 적합7 = 채택, ★비직관적 픽
- ☠️ 함정: "신호등 앱 주문" → 앱 개발비 3천만원, 손님 50명 라면집엔 과잉
- 📦 군집화: [속도 계열] [기억 계열] [신뢰 계열]

### 한 문장 요약

> **AI 한 명에게 5번 묻는 게 아니라, 서로 격리된 AI 5명을 동시에 굴리고, 그 결과를 6번째 AI(편집장)가 채점·군집화·함정표시하는 장치.**

### 킬러 기능: 함정(trap) 목록

보통 AI는 전부 칭찬한다. ADHD는 **"예뻐 보이는데 하면 망합니다. 이유는 이것입니다"**를 이름 붙여 따로 빼준다. 평가에서 5.2배 차이 난 항목.

심판의 실제 평:
> *"B의 함정 섹션은 내가 며칠 날렸을 18개 막다른 길에서 구해줬다. `shelve`가 스레드 안전하지 않은 걸 몰랐고, Redis를 먼저 시도했을 것이다."*

### 닻(anchor) 뽑기

> "우리 **MySQL** 쓰는데 조회가 느려. 개선 방법?"

일반 AI → MySQL 인덱스 얘기만. ("MySQL"이라는 닻에 묶임)

ADHD Phase 0 → "조회 응답시간을 줄여야 한다" (DB 이름 제거)
→ "아예 DB 안 쓰고 메모리에?", "읽기를 미리 계산?", "조회 자체를 없애면?"

단, "금융규제 때문에 감사로그 필수" 같은 **진짜 제약은 보존.**

---

## 5. 설치 및 사용법

### 방법 A — 스킬로 (권장)

```bash
npx skills add UditAkhourii/adhd
```

`skills` CLI(Vercel Labs)가 사용 중인 에이전트를 자동 감지해서 `SKILL.md`를 알맞은 위치에 배치.
지원: Claude Code, Claude.ai, Cursor, Codex, Cline, Continue, Aider, Gemini CLI, Windsurf, Cody, Roo, Augment, OpenCode, Kilo, Kimi, Qwen, Trae, Replit, Warp 등 **약 50개**

```bash
npx skills add UditAkhourii/adhd -g                        # 전역 설치
npx skills add UditAkhourii/adhd -a claude-code -a cursor  # 특정 에이전트만
npx skills add UditAkhourii/adhd --copy                    # 심볼릭링크 대신 복사
npx skills add UditAkhourii/adhd --list                    # 내용 확인
```

수동 설치:

```bash
# Claude Code (전역)
mkdir -p ~/.claude/skills/adhd
curl -fsSL https://raw.githubusercontent.com/UditAkhourii/adhd/main/skills/adhd/SKILL.md \
  -o ~/.claude/skills/adhd/SKILL.md

# Claude Code (프로젝트별)
mkdir -p .claude/skills/adhd
curl -fsSL https://raw.githubusercontent.com/UditAkhourii/adhd/main/skills/adhd/SKILL.md \
  -o .claude/skills/adhd/SKILL.md

# Cursor
curl -fsSL https://raw.githubusercontent.com/UditAkhourii/adhd/main/skills/adhd/SKILL.md >> .cursorrules

# Codex (일부 빌드는 강제 지정 필요)
npx skills add UditAkhourii/adhd -a codex -g
```

Claude.ai 웹/데스크톱: 프로젝트 설정 → Skills → Add skill → `SKILL.md` 업로드

사용: `/adhd "리더 선출 중에도 정확한 레이트 리미터 설계해줘"` 또는 브레인스토밍 의도로 자동 트리거

### 방법 B — CLI

```bash
npm install -g adhd-agent
adhd "리더 선출 중에도 정확한 레이트 리미터 설계해줘"
adhd "이 함수 이름 뭐로 할까" --frames 3 --ideas 8 --top 2
adhd "..." --context ./client.ts --json > out.json
```

| 플래그 | 기본값 | 설명 |
|---|---|---|
| `--frames N` | 5 | 병렬 발산 브랜치 수 (최대 20) |
| `--ideas N` | 6 | 브랜치당 아이디어 수 (최대 50) |
| `--top N` | 3 | 심화할 개수 (최대 20) |
| `--concurrency N` | 4 | 최대 동시 LLM 호출 (최대 16) |
| `--context PATH` | — | 파일을 컨텍스트로 주입 (최대 10MB) |
| `--model NAME` | SDK 기본 | 생성기+비평가 모델 |
| `--critic-model N` | =`--model` | **비평가만 다른 모델로** → 오류 상관관계 제거 |
| `--pack NAME` | `core` | 프레임 팩 (반복 가능, 풀링) |
| `--no-code-mode` | — | 엔지니어링 편향 끄기 |
| `--no-anchor-strip` | — | Phase 0 닻 제거 끄기 |
| `--json` | — | `RunResult` JSON 출력 |
| `--quiet` | — | 진행상황 숨김 |

### 방법 C — 라이브러리

```bash
npm install adhd-agent
```

```ts
import { run, renderText } from "adhd-agent";

const result = await run({
  problem: "버스트 부하에서 큐를 어떻게 샤딩할까?",
  framesPerRun: 5,
  topK: 3,
  onEvent: (e) => console.log(e.kind),  // 진행상황 스트리밍
});

console.log(renderText(result));
// result.shortlist · result.nonObviousPick · result.traps
// · result.deepened · result.clusters
```

### 방법 D — Claude Code 플러그인

`.claude-plugin/marketplace.json`이 있어 플러그인 마켓플레이스로도 설치 가능 (커밋 `3d9dc48`).

---

## 6. 플러그인? 스킬? MCP?

**결론: 스킬이 본체 + 플러그인 포장 + npm 패키지. MCP는 아니다.**

| 형태 | 존재 | 근거 |
|---|:---:|---|
| **Skill** | ✅ **본체** | `skills/adhd/SKILL.md` (215줄, YAML frontmatter + 본문) |
| **Plugin** | ✅ 포장 | `.claude-plugin/plugin.json` + `marketplace.json` |
| **npm 패키지** | ✅ 별도구현 | `package.json` → `adhd-agent` v0.1.4, bin: `adhd` |
| **MCP 서버** | ❌ **아님** | MCP 관련 파일/의존성 전혀 없음 |

### MCP가 아닌 이유

MCP는 "AI에게 외부 도구/데이터 접근권을 주는 프로토콜"(stdio/SSE 서버, tool 노출)이다.
ADHD는 그 반대로, **도구를 의도적으로 비워둔다:**

```ts
// src/llm.ts
tools: [] as string[],
// 주석: "No tools — divergence is pure generation.
//        Tools = convergence pressure."
```

**ADHD는 "AI가 생각하는 방식(사고 절차)"을 바꾸고, MCP는 "AI가 만질 수 있는 것(권한)"을 늘린다.** 완전히 다른 레이어.

### 스킬 vs 라이브러리

| | SKILL.md | src/ (npm) |
|---|---|---|
| 실행자 | **Claude 자신** (Task/Agent 툴로 팬아웃) | Node 프로세스 |
| 의존성 | **없음** (마크다운 1개) | `@anthropic-ai/claude-agent-sdk`, `p-limit`, `zod` |
| 설치 | 파일 복사 | `npm i` |
| JSON 검증 | Claude가 알아서 | Zod 스키마 엄격 검증 |
| 용도 | 대화형 개발 | 배치 · CI · 앱 임베딩 |

같은 루프를 두 번 구현한 것. SKILL.md 원문:
> *"The skill above gives you the same loop inside Claude with no install required."*

---

## 7. API 토큰 필요한가?

| 사용 형태 | 토큰 | 상세 |
|---|:---:|---|
| **SKILL.md만 설치** | ❌ 불필요 | Claude Code/Claude.ai 구독으로 끝 |
| **CLI (`adhd`)** | ⚠️ 조건부 | `ANTHROPIC_API_KEY`, **또는** 로컬 Claude Code 인증 상속 |
| **라이브러리** | ⚠️ 조건부 | 동일 (Agent SDK가 자동 탐색) |
| **다른 에이전트** | ❌ 불필요 | 해당 에이전트 구독 사용 |

문서 원문:
> *"Auth: picks up `ANTHROPIC_API_KEY` from the environment, or inherits auth from a local Claude Code install."*

### 비용 (문서가 매우 솔직함)

```
cost ≈ N × (base_context + branch_output)   ← 발산
     + critic_context                        ← 채점+군집 (N×k 아이디어 전부)
     + K × deepen_context                    ← 심화
```

> **"'10번 호출'이라는 프레이밍이 숨기는 건 `base_context`에 붙은 `N ×` 승수다."**

브랜치마다 신선한 격리 컨텍스트라서 `CLAUDE.md`와 툴 컨텍스트가 매번 재로드된다.
base substrate가 26K 토큰이면 브랜치 5개 = 아이디어 1토큰 생성 전에 이미 **130K 토큰** 소모.

| 사용법 | 비용 |
|---|---|
| CLI/라이브러리 | substrate 작음 → 단발의 **5~10배** |
| Claude Code 스킬 | substrate 큼 → **훨씬 높음**, N에 비례 증가 |

> *"고위험 결정 하나를 넓히는 데 몇 센트~몇 달러. 잘못된 뻔한 답을 배포하는 것보다 싸다. 매 키스트로크마다 돌리지 마라. 의사결정 지점에서 돌려라."*

### 프리플라이트 게이트 (`SKILL.md`)

스킬이 스스로 비용을 알고 거부한다:

1. **명시적 호출 체크** — `/adhd` 입력했으면 게이트 통과, 재고하지 않음
2. **자체 판단** (명시 호출이 아닐 때) — 셋 중 하나라도 No면 ABORT
   - 열린 문제인가? (정답이 하나면 중단)
   - 고위험인가? (뻔한 답이 틀렸을 때 비용이 큰가)
   - 열린 표현인가? ("빠르게", "표준", "일반적인", "그냥", "한줄" 있으면 중단)

---

## 8. 왜 깃허브에서 유명한가

### 1. 네이밍
**"ADHD"** — 3글자, 즉시 발음·이해되고 약간 도발적. 밈 친화적. "ADHD for coding agents"는 한 번 들으면 안 잊힌다.

### 2. 진입장벽 0
`npx skills add UditAkhourii/adhd` — 마크다운 파일 1개. 빌드·의존성·서버·API키 없음.
*"읽고 두 개의 다른 에이전트에 설치했다"* 는 후기가 나오는 이유.

### 3. "프롬프트 팁"이 아니라 "논문"으로 포지셔닝
- 프리프린트: *"ADHD: Parallel Divergent Ideation for Coding Agents"*
- 평가 하네스 공개: 6문제 + 독립 LLM 심판 + A/B 랜덤화 + 152KB 트랜스크립트
- **1패를 숨기지 않음** (`llm-hang-cli`)

### 4. 트윗하기 좋은 숫자
> **함정 탐지 9.5 vs 1.8 = 5.2배**

### 5. 미디어 + 소셜 증명
- The New Stack 피처 기사
- Trendshift 일간 트렌딩 뱃지 (#39300)
- Discord 커뮤니티 (프레임 설계, 함정 사냥)
- ADOPTERS.md에 **17개+ 프로젝트**: repowire(PR #313 머지), mstack, zk-flow-oss, han, wtfismyrepo, awesome-prompts, nix-skills, multi, striatum, mythify, godaudits, godplans 등
- 독립 리뷰 2건 (han 증거기반 리서치 11출처/8검증라운드, 일본 블로거 블라인드 벤치마크)

### 6. 비판을 이슈로 공개 추적
`han` 팀의 학술적 반박(프레임 vs 페르소나 연구 혼동 등)을 저자가 **issues #16~#18로 공개 등록**하고 README에 링크까지 걸었다. 역설적으로 신뢰를 폭발시키는 무브.

### 7. 타이밍
2026년은 "에이전트 스킬" 표준이 자리잡는 시기. 그 순간에 "50개 에이전트 전부 지원"을 들고 나왔다.

---

## 9. 로컬 에이전트 구축에 도움되는가 → 매우 그렇다

### 층1: 교재로서 (가장 큰 가치)

`src/`는 **에이전트 오케스트레이션 레퍼런스 구현체**다.

| 패턴 | 파일 | 배울 것 |
|---|---|---|
| 병렬 팬아웃 + 세마포어 | `engine.ts` | `pLimit(concurrency)` + `Promise.all` |
| 격리 컨텍스트 강제 | `llm.ts` | `query()` 호출마다 stateless. KV캐시·히스토리 공유 0 |
| 시스템 프롬프트 반전 | `engine.ts` | `DIVERGE_SYSTEM` ↔ `SCORE_SYSTEM` 성격 반전 |
| 구조화 출력 + Zod | `engine.ts` | `DivergeRowSchema`, `ScoreRowSchema`, `ClusterSchema` |
| Graceful degradation | `engine.ts` | 파싱 실패 시 `catch { return [] }` — 브랜치 1개 죽어도 런은 산다 |
| LLM 출력 방어 파싱 | `llm.ts` | ` ```json ` 펜스 벗기고 첫 `{`/`[` 찾기 |
| 이벤트 스트리밍 | `types.ts` | `RunEvent` 유니온 타입 → CLI 진행표시 |
| 비평가 모델 분리 | `engine.ts` | `criticModel ?? model` — 다른 모델패밀리로 오류 상관관계 제거 |
| 플러그형 데이터 | `frames.ts` | `PACKS` 레코드에 추가만 하면 `selectFrames` 안 건드리고 확장 |
| 비용 게이트 | `SKILL.md` | 프리플라이트로 스스로 ABORT |

### 층2: 에이전트 안에 부품으로 심기

```
[사용자 요청]
   → [의도 분류]
   → 열린 설계 문제? ─YES→ ADHD.run() → shortlist/traps
   │                                         ↓
   └─NO→ 단발 응답                    [사람이 고름] → [실행]
```

`wtfismyrepo`가 정확히 이렇게 했다: 결정론적 분석(import 그래프 PageRank, git churn, PR 신호) → 분석 객체 → ADHD `problem` + `context` → 12개 코드베이스 전용 프레임(`new-grad`, `archeologist`, `security-researcher`, `on-call-at-3am`, `refactorer`, `inversion`…) → 온보딩 각도 생성.

### 커스텀 프레임팩 확장

```ts
// src/packs/korean-web.ts
export const koreanWeb: Frame[] = [
  { id: "naver-seo", label: "네이버 SEO 담당자",
    prompt: "네이버 검색 노출 관점으로 재질문. 블로그·카페·지식iN이 준 신호는?",
    tags: ["design", "general"] },
  { id: "kakao-share", label: "카카오톡 공유 설계자",
    prompt: "이게 단톡방에 공유될 때를 상상해. OG태그·썸네일·1줄요약이 뭐가 되어야?",
    tags: ["design", "wild"] },
  { id: "pc-bang", label: "PC방 사장님",
    prompt: "100대 동시접속, 사양 낮은 구형PC, 인터넷은 빠름. 뭐가 병목이고 뭘 버려야?",
    tags: ["code", "wild"] },
];
```

```ts
// src/frames.ts
export const PACKS: Record<string, Frame[]> = { core, koreanWeb };
```

```bash
adhd "..." --pack core --pack koreanWeb   # 풀링됨
```

### 주의점 / 발견된 이슈

1. **Claude Agent SDK 강결합** — `src/llm.ts`가 `@anthropic-ai/claude-agent-sdk`의 `query()`를 직접 사용. OpenAI/Gemini/로컬LLM을 쓰려면 어댑터 추상화 필요. (`multi` 프로젝트가 크로스모델로 포팅함)
2. **문서-코드 불일치 (기여 기회)** — `documentation/how-it-works.md`가 `src/diverge.ts`, `src/score.ts`, `src/cluster.ts`, `src/deepen.ts`를 참조하지만 **그 파일들은 존재하지 않는다.** 전부 `src/engine.ts` 안에 있음. 리팩토링 후 문서 미갱신.
3. `src/cli.ts` 최상단 주석이 아직 구 이름 `connect-dots`.
4. `README.md` 최상단 `<p align="center">` 여는 태그 누락 (`</p>`만 존재).

---

## 10. React / PHP로 만들 수 있는가 → 가능

ADHD는 알고리즘이 아니라 **오케스트레이션 패턴**이다. 필요한 건 3개뿐:

```
① HTTP로 LLM API 호출   ② 병렬 실행   ③ JSON 파싱
```

### React

⚠️ **브라우저에서 API키를 직접 쓰면 절대 안 된다.** 브라우저 코드는 누구나 볼 수 있다.

```
[React] ──fetch──> [백엔드: Next.js API Route / PHP / Node]
                         └─ 여기서 API키 보관 + ADHD 루프 실행
```

React의 강점은 **시각화**:

```jsx
function AdhdBoard({ result }) {
  return (
    <div className="grid gap-6">
      {/* 클러스터별 칸반 */}
      <section className="grid grid-cols-3 gap-4">
        {result.clusters.map(c => (
          <div key={c.label} className="rounded-xl border p-4">
            <h3 className="font-bold text-cyan-600">{c.label}</h3>
            {c.ideaIds.map(id => {
              const i = findIdea(result, id);
              return (
                <IdeaCard key={id} idea={i}>
                  <ScoreRadar n={i.score.novelty}
                              v={i.score.viability}
                              f={i.score.fit} />
                </IdeaCard>
              );
            })}
          </div>
        ))}
      </section>

      <HeroCard badge="★ 비직관적 픽" idea={result.nonObviousPick} />
      <TrapList traps={result.traps} />
      {result.deepened.map(d => (
        <Accordion key={d.ideaId} sketch={d.sketch} children={d.childIdeas} />
      ))}
    </div>
  );
}
```

SSE(Server-Sent Events)로 `RunEvent`를 실시간 스트리밍하면 프레임 완료가 하나씩 표시된다.
`src/types.ts`의 `RunEvent` 유니온이 이미 그 용도로 설계되어 있다.

### PHP (8.1+, `curl_multi`로 병렬)

```php
<?php
declare(strict_types=1);

const DIVERGE_SYSTEM = <<<'TXT'
You are in DIVERGENT mode. You are a generator, not a critic.
- Output a JSON array only. No prose before/after.
- Each idea is a SHORT phrase. No paragraphs.
- The first 3 obvious ideas are banned. Aim for the awkward middle.
- Do not evaluate, hedge, or rank. Just generate.
TXT;

const FRAMES = [
  ['id'=>'hardware-eyes', 'label'=>'하드웨어 엔지니어',
   'prompt'=>'레이턴시·메모리 레이아웃·물리적 제약으로 생각한다. 이 문제를 하드웨어/펌웨어 문제로 재질문하라.',
   'tags'=>['code','wild']],
  ['id'=>'biology', 'label'=>'생물학',
   'prompt'=>'면역계·신경가소성·세포신호·진화·장내미생물에서 메커니즘을 가져와 이 공학 문제에 억지로 이식하라.',
   'tags'=>['code','wild']],
  ['id'=>'speedrunner', 'label'=>'스피드러너',
   'prompt'=>'글리치·스킵·아웃오브바운드 꼼수를 찾아라. 악용적이지만 합법인 경로는?',
   'tags'=>['code','wild']],
  // ... 나머지 12개
];

/** Phase 1 — 병렬 발산 (curl_multi = TS의 Promise.all) */
function diverge(string $problem, array $frames, int $ideasPerFrame = 6): array {
    $mh = curl_multi_init();
    $handles = [];

    foreach ($frames as $frame) {
        $body = json_encode([
            'model'      => 'claude-opus-5',
            'max_tokens' => 2048,
            'system'     => DIVERGE_SYSTEM . "\n\nFRAME — {$frame['label']}:\n{$frame['prompt']}",
            'messages'   => [[
                'role'    => 'user',
                'content' => "PROBLEM:\n{$problem}\n\n"
                           . "Generate {$ideasPerFrame} ideas under this frame.\n"
                           . 'Output JSON array: [{"text":"...","rationale":"..."}]',
            ]],
        ], JSON_UNESCAPED_UNICODE);

        $ch = curl_init('https://api.anthropic.com/v1/messages');
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_TIMEOUT        => 120,
            CURLOPT_HTTPHEADER     => [
                'content-type: application/json',
                'x-api-key: ' . getenv('ANTHROPIC_API_KEY'),   // 하드코딩 금지
                'anthropic-version: 2023-06-01',
            ],
            CURLOPT_POSTFIELDS     => $body,
        ]);
        curl_multi_add_handle($mh, $ch);
        $handles[$frame['id']] = $ch;   // 격리 지점: 서로의 응답을 절대 넘기지 않음
    }

    do {
        $status = curl_multi_exec($mh, $running);
        if ($running) curl_multi_select($mh, 1.0);
    } while ($running && $status === CURLM_OK);

    $branches = [];
    foreach ($handles as $frameId => $ch) {
        $raw = curl_multi_getcontent($ch);
        curl_multi_remove_handle($mh, $ch);
        curl_close($ch);
        $branches[$frameId] = parseIdeas($raw);   // 실패해도 [] 반환
    }
    curl_multi_close($mh);
    return $branches;
}

/** llm.ts의 parseJSON 포팅 — LLM은 항상 ```json으로 감싼다 */
function parseIdeas(string $raw): array {
    $env  = json_decode($raw, true);
    $text = $env['content'][0]['text'] ?? '';
    if (preg_match('/```(?:json)?\s*([\s\S]*?)```/', $text, $m)) $text = trim($m[1]);
    $s = strpos($text, '['); $o = strpos($text, '{');
    $start = ($s === false) ? $o : (($o === false) ? $s : min($s, $o));
    if ($start > 0) $text = substr($text, $start);
    $ideas = json_decode($text, true);
    return is_array($ideas) ? $ideas : [];
}

/** 가중 점수 — engine.ts와 동일 */
function weighted(array $s): float {
    return $s['novelty'] * 0.35 + $s['viability'] * 0.40 + $s['fit'] * 0.25;
}
```

### 언어 무관 필수 불변식 5가지

하나라도 어기면 ADHD가 아니라 "넓은 단일 생각"이 된다 (`SKILL.md` 안티패턴 섹션):

| # | 불변식 | 위반 시 |
|:-:|---|---|
| 1 | **브랜치는 반드시 병렬 + 격리** | 순차 실행/출력 전달 → 앵커링 부활, 방법론 붕괴 |
| 2 | **발산 단계엔 비평가 0** | 생성기가 목졸림. "평가 금지" 시스템 프롬프트 필수 |
| 3 | **생성기/비평가 = 물리적으로 다른 호출** | 한 프롬프트에 "생성하고 평가해"는 가짜. 기계적 분리가 생명 |
| 4 | **발산 후 반드시 수렴** | 정렬 안 된 30개 잡탕은 안전한 1개 답만큼 쓸모없음 |
| 5 | **입장을 취해라** | "20개 줬으니 알아서 골라"는 도망 |

### 언어별 비교

| 언어 | 난이도 | 강점 |
|---|:---:|---|
| TypeScript | ⭐ | 원본 그대로. Zod 검증 |
| React (+백엔드) | ⭐⭐ | **시각화 압도적.** SSE 실시간 + 칸반 + 레이더차트 |
| PHP | ⭐⭐ | `curl_multi`로 충분. 워드프레스/라라벨 생태계 |
| Python | ⭐ | `asyncio.gather` — 가장 쉬움 |

**추천 조합: Next.js(API Route로 키 보호 + 루프 실행) + React(시각화)**

이유: 현재 ADHD의 가장 큰 약점이 **"결과를 보기 어렵다"**는 것. `EVALS.md`의 심판도
*"B는 파싱하기 어렵지만 결정에 중요한 정보가 더 많다"* 고 적었다.
"정보는 좋은데 읽기 힘들다" = 파고들 틈.

---

## 11. 수익화 아이디어

**전제: MIT 라이선스 — 상업적 사용 가능, 저작자 표시 필수.**

### 🥇 TIER 1 — 지금 당장, 혼자, 저비용

#### 1) 한국 시장 전용 프레임팩 `adhd-kr` (1순위 추천)

15개 기본 프레임은 전부 서구권 엔지니어링 관점이다.

```ts
// src/packs/korea.ts
export const korea: Frame[] = [
  { id: "naver-seo", label: "네이버 SEO 담당자",
    prompt: "구글이 아니라 네이버 검색 노출 관점으로 재질문하라. 블로그·카페·지식iN·뷰탭이 주는 신호는? VIEW 최적화는?",
    tags: ["design", "general"] },

  { id: "kakao-viral", label: "카카오톡 단톡방 공유 설계자",
    prompt: "이게 단톡방에 공유되는 순간을 상상하라. OG 썸네일·1줄 요약·미리보기가 뭐가 되어야 눌리는가?",
    tags: ["design", "wild"] },

  { id: "toss-ux", label: "토스 UX 원칙주의자",
    prompt: "화면당 액션 1개, 전문용어 0개. 이 문제를 '한 화면 한 동작'으로 쪼개면? 없앨 수 있는 단계는?",
    tags: ["design", "general"] },

  { id: "coupang-scale", label: "쿠팡 트래픽 엔지니어",
    prompt: "새벽배송 마감 10분 전 스파이크를 가정하라. 뭐가 먼저 터지고 뭘 버려야 서비스가 사는가?",
    tags: ["code", "design"] },

  { id: "pc-bang", label: "PC방 사장님",
    prompt: "구형 PC 100대, 동시접속, 인터넷만 빠름. 클라이언트 사양이 바닥일 때 뭘 서버로 옮기는가?",
    tags: ["code", "wild"] },

  { id: "kisa-audit", label: "KISA/개인정보위 심사관",
    prompt: "개인정보보호법·정보통신망법 관점. 주민번호·본인인증·위치정보 처리에서 증명해야 할 것은? 파기 절차는?",
    tags: ["design", "general"] },

  { id: "ajumma-test", label: "우리 엄마 테스트",
    prompt: "스마트폰은 쓰지만 앱 설치는 자녀가 해주는 사용자. 이 기능을 설명 없이 쓰게 하려면?",
    tags: ["general", "wild"] },

  { id: "military-net", label: "군 인트라넷 개발자",
    prompt: "외부망 차단, npm 불가, 승인된 라이브러리만. 의존성 0으로 이걸 만들면?",
    tags: ["code", "wild"] },
];
```

| 단계 | 내용 | 예상 수익 |
|---|---|---|
| 1. 무료 OSS | GitHub 공개 + ADOPTERS.md PR → 역링크 | ₩0 (인지도 = 자산) |
| 2. 유료 확장팩 | 핀테크팩 / 이커머스팩 / 게임팩 (Gumroad, Lemon Squeezy) | ₩15,000~50,000/팩 |
| 3. 컨설팅 유입 | "프레임팩 만든 사람" 포지션 | ₩300만~1,000만/건 |

**강점**: 코드를 거의 안 짜도 된다(파일 1개 + `PACKS` 한 줄 등록). 진입장벽은 "한국 IT 도메인 지식"이라 외국인이 못 만든다.

#### 2) 한국어 번역 + 콘텐츠 자산화

| 산출물 | 수익화 |
|---|---|
| `README.ko.md` + 8개 문서 번역 → 원본에 PR | 원본 리포에 이름 영구 등록 |
| 티스토리/벨로그 시리즈 "AI 에이전트 발산적 사고" 5부작 | 애드센스 + 개인브랜딩 |
| 유튜브 "AI가 뻔한 답만 하는 이유" | 조회수 + 강의 유입 |
| 인프런/클래스101 강의 "Claude Code 스킬 만들기 실전" | ₩50,000~99,000 × 수강생 |
| 전자책 "에이전트 오케스트레이션 패턴 10개" | 부크크/리디 |

부가 인사이트: 이 리포 자체가 *"오픈소스를 어떻게 유명하게 만드는가"*의 교과서다.
프리프린트 + 평가하네스 + ADOPTERS + Discord + 미디어 + 비판 공개추적 — 이 **성장 플레이북 자체가 콘텐츠 상품**이 된다.

#### 3) 컨설팅 / 워크숍

| 상품 | 구성 | 가격대 |
|---|---|---|
| 반나절 워크숍 | 발산/수렴 이론 + 실습 + 사내 프레임팩 설계 | ₩150만~400만 |
| 커스텀 프레임팩 개발 | 도메인 전용 프레임 12~15개 | ₩500만~1,500만 |
| 에이전트 파이프라인 구축 | ADHD 루프 사내 통합 | ₩2,000만~ |
| 리테이너 | 월간 의사결정 리뷰 + 프레임 튜닝 | ₩200만/월 |

세일즈 무기: *"함정 탐지 5.2배"* + *"우리 팀이 3일 날린 삽질을 미리 알려줍니다"*

---

### 🥈 TIER 2 — 개발 필요, 차별화 큼

#### 4) 시각화 SaaS `DivergeBoard` (제품성 최고)

**문제**: ADHD 결과가 읽기 어렵다. 터미널에 30개 아이디어 + 20개 함정 + 3개 심화가 ANSI 컬러로 쏟아진다.

```
┌─────────────────────────────────────────────────────┐
│ 🎯 문제: 리더 선출 중에도 정확한 레이트 리미터       │
│ ↺ 닻 제거됨: "Redis" → "분산 카운터"                │
├─────────────────────────────────────────────────────┤
│ ▸ 하드웨어 ✓6  ▸ 생물학 ✓6  ▸ 개미집단 ⏳ ...      │  ← SSE 실시간
├──────────────┬──────────────┬───────────────────────┤
│ 서버제거 계열 │ 캐시형 계열   │ 레이스형 계열         │  ← 클러스터 칸반
│ ┌──────────┐ │ ┌──────────┐ │ ┌──────────┐          │
│ │아이디어   │ │ │아이디어   │ │ │아이디어   │          │
│ │ ◢N8V7F9  │ │ │ ◢N5V9F8  │ │ │ ◢N9V4F6  │          │  ← 레이더 차트
│ │ [드래그] │ │ │          │ │ │ ☠️ 함정   │          │
│ └──────────┘ │ └──────────┘ │ └──────────┘          │
├──────────────┴──────────────┴───────────────────────┤
│ ★ 비직관적 픽: "느린 모델이 애초에 잘못된 모델"      │
│ ☠️ 함정 20개 [펼치기] — 각각 이유 1줄                │
│ 💬 팀 투표: 👍3 👎1  |  💭 코멘트 5                  │
└─────────────────────────────────────────────────────┘
```

원본에 없는 차별 기능:
- SSE 실시간 진행 (`RunEvent` 스트리밍 — 타입 이미 설계됨)
- 클러스터 칸반 보드 (드래그 재분류)
- 점수 레이더차트 (N/V/F 3축)
- **팀 투표 + 코멘트** — 혼자 쓰는 도구를 팀 도구로 격상 (B2B 가격 근거)
- 히스토리 & 재실행 (같은 문제 재실행 → 다른 후보군 비교)
- Notion/Slack/Linear 내보내기
- 베이스라인 vs ADHD A/B 비교 뷰

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | ₩0 | 월 5회, BYOK |
| Pro | **$19/월** | 무제한, 히스토리, 내보내기 |
| Team | **$49/유저/월** | 투표, 코멘트, 결정로그, SSO |
| Enterprise | 협의 | 온프레미스, 커스텀 프레임팩 |

**BYOK(Bring Your Own Key) 전략이 핵심** — LLM 비용을 고객이 부담하므로 원가 ≈ 0.

#### 5) 함정탐지 특화 `TrapGuard`

논리: 평가에서 **함정탐지만 5.2배** 차이(다른 항목은 1.1~2.9배). 가장 강한 무기인데 아무도 단독 상품화하지 않았다.

```
[PR 열림]
  → 변경 파일 + 설계 의도 추출
  → ADHD 발산 (adversary / regulator / ops-3am / remove-assumption 프레임)
  → 비평가가 함정만 필터링
  → PR 코멘트:

  ☠️ 함정 3개 감지 (이 PR이 채택한 접근의 숨은 비용)

  1. 낙관적 락 → 동시 쓰기 10k/s 넘으면 재시도 폭풍.
     프로토타입엔 충분하지만 현재 트래픽 기준 3개월 후 한계.
  2. 인메모리 카운터 → 리더 재선출 시 전체 리셋.
     레이트리밋이 무력화되는 창이 열림.
  3. 조기 추상화: Strategy 패턴 도입했으나 구현체 1개.
     제거하면 코드 40줄 감소.

  ✅ 잘한 점: 트랜잭션 경계가 명확함
```

왜 팔리는가:
- "아이디어 더 주세요" = nice to have
- "너 지금 함정에 빠지려 해" = **must have**
- ROI 증명 쉬움: *"이번 달 함정 12개 감지 = 엔지니어 40시간 절약"*

형태: GitHub App / GitLab / Bitbucket · 가격: $10/리포/월 또는 $8/개발자/월

#### 6) `adhd-php` / Laravel 패키지 (빈 시장)

현재 생태계는 전부 Node/TS. 그런데:
- 워드프레스 = 웹의 43%
- 라라벨 = PHP 프레임워크 1위
- 한국 중소 SI/에이전시 = PHP 다수

```
composer require bmshin/adhd-php
```

| 산출물 | 수익 |
|---|---|
| `adhd-php` (Composer, MIT) | 무료 → 인지도 |
| Laravel 패키지 (Artisan 명령, Queue 연동, Nova 위젯) | 무료 → 유입 |
| **워드프레스 플러그인** "AI 기획 브레인스토머" | **$49** (Envato/CodeCanyon) |
| PHP 에이전시용 화이트라벨 | ₩500만~ |

워드프레스 플러그인이 핵심 노림수: CodeCanyon "AI + 워드프레스" 카테고리 경쟁자가 대부분 단순 ChatGPT 래퍼. "구조화된 발산적 사고"는 없다.

---

### 🥉 TIER 3 — 장기, 큰 판

#### 7) 프레임팩 마켓플레이스

`.claude-plugin/marketplace.json`이 이미 있다는 게 힌트. "프레임의 앱스토어".

```
DivergeHub — 프레임팩 마켓
├── 🏥 의료기기 규제팩 (FDA/MFDS/CE)        $79
├── 🎮 게임 라이브서비스팩                   $59
├── 💰 핀테크 컴플라이언스팩 (전금법/PCI)    $99
├── 🔐 보안 레드팀팩                        $89
├── 🇰🇷 한국 웹서비스팩 (무료·리드용)        $0
└── 🏭 제조 IoT팩                           $69
```

수수료 30% + 자체 프리미엄팩 직판.

왜 통하는가: 프레임은 **도메인 전문성의 결정체**다. 의료기기 규제 전문가는 지식이 있지만 코드는 못 짠다 → 플랫폼이 되면 전문가는 지식만, 운영자는 유통을 맡는다.

#### 8) 의사결정 기록 시스템 `DecisionLog`

기업의 진짜 고통: *"이 아키텍처 왜 이렇게 됐지? 결정한 사람 퇴사했는데 기록이 없어."*

```markdown
# ADR-042: 이벤트 소싱 대신 CDC 채택

## 검토 시점: 2026-09-27
## 검토된 각도: 5개 프레임, 아이디어 30개

### 채택: CDC (Change Data Capture)  [N6 V9 F9]
근거: 기존 DB 스키마 유지 + 점진 마이그레이션 가능

### 검토했으나 탈락 (☠️ 함정)
- 이벤트 소싱 전면 도입 → 팀에 경험자 0명, 6개월 학습비용
- 듀얼 라이트 → 정합성 검증 불가, 조용히 갈라짐
- 트리거 기반 → DB 부하 2배, 벤더락인

### ★ 비직관적이었지만 검토한 것
- "애초에 동기화를 없앤다" → 읽기 모델을 클라이언트가 조립
  (탈락 이유: 모바일 구버전 지원 불가)
```

ISO 27001 / SOC2 심사의 *"의사결정 근거를 문서화했는가"* 항목에 그대로 제출 가능 → B2B 가격 정당화 ($99~299/월/팀).

---

### 실행 로드맵

| 기간 | 할 일 | 목표 |
|---|---|---|
| 1~2주 | ① 한국어 번역 PR ② 문서-코드 불일치 수정 PR (존재하지 않는 `src/diverge.ts` 등 참조) | 원본 기여자 등록 |
| 3~4주 | `adhd-kr` 프레임팩 OSS 공개 + ADOPTERS.md PR | 역링크 + 한국 커뮤니티 인지도 |
| 2개월 | 블로그 5부작 + `adhd-php` Composer 패키지 | 콘텐츠 자산 + 빈 시장 선점 |
| 3~4개월 | `DivergeBoard` MVP (Next.js + React, BYOK, SSE, 칸반) | 첫 유료 고객 |
| 6개월 | `TrapGuard` GitHub App | 반복 매출(MRR) |
| 1년 | 프레임팩 마켓 또는 워크숍 사업 | 스케일 |

### 리스크와 대응

| 리스크 | 대응 |
|---|---|
| 원본이 직접 UI를 만들 경우 | 팀 협업(투표·코멘트·결정로그)으로 차별화. 저자는 연구/방법론 포지션이라 협업 SaaS 가능성 낮음 |
| 모델사가 기본 기능화 | 도메인 프레임팩은 모델사가 만들지 않는 영역 |
| MIT라 카피 가능 | 브랜드 + 데이터(어떤 프레임이 잘 먹히는지) + 커뮤니티가 방어막 |
| LLM 비용 부담 | BYOK 전략으로 원가 ≈ 0 |

### 결론

> 원본은 **"방법론"**을 만들었다. 차별화 지점은 그 방법론의 **"인터페이스"**와 **"도메인 지식"**이다. 둘은 경쟁이 아니라 보완 관계다.
>
> 특히 **한국 프레임팩**(네이버·카카오·토스·쿠팡·PC방·군인트라넷·엄마테스트)은 외국 개발자가 만들 수 없다. **가장 작은 노력으로 가장 대체 불가능한 자산**이 되는 지점.

---

## 12. 요약 한 장

| 질문 | 답 |
|---|---|
| 뭐하는 건가? | LLM의 조급한 수렴을 막는 **병렬 발산 이데이션** 장치 |
| 어떻게? | N개 격리 브랜치를 서로 다른 인지 프레임으로 병렬 실행 → 별도 비평가가 채점·군집·함정탐지·심화 |
| 언제 쓰나? | 아키텍처/API 설계, 네이밍, 퍼지 디버깅, 전략, "몇 가지 방법 줘봐" 형태 |
| 언제 안 쓰나? | 문법, 원인 아는 버그, 구글 1번, 매 키스트로크 |
| 형태는? | **스킬**(본체) + 플러그인 포장 + npm 패키지. **MCP 아님** |
| 토큰 필요? | 스킬로 쓰면 불필요. CLI/라이브러리는 `ANTHROPIC_API_KEY` 또는 Claude Code 인증 상속 |
| 비용? | 단발의 5~10배 (스킬은 더 높음, N에 비례) |
| 최고 가치? | **함정 탐지 5.2배** = 삽질 방지 |
| 유명한 이유? | 네이밍 + 진입장벽 0 + 논문 포지셔닝 + 트윗 가능한 숫자 + 미디어/채택자 + 비판 공개추적 + 타이밍 |
| 다른 언어로? | 가능. 5가지 불변식(병렬격리·비평가OFF·기계적분리·반드시수렴·입장취하기)만 지키면 됨 |
| 최대 기회? | 결과 시각화 UI + 한국 도메인 프레임팩 |

---

*이 문서는 `bmshin94/adhd` (https://github.com/bmshin94/adhd) 리포지토리 전수조사 결과입니다.*
*원본: https://github.com/UditAkhourii/adhd (MIT License, © Udit Akhouri)*
