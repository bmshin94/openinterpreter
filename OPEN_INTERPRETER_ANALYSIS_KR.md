# Open Interpreter (Rust) 전수조사 분석 및 활용·수익화 정리

> 이 문서는 `bmshin94/openinterpreter` 저장소를 폴더 단위로 전수조사한 결과와,
> 이어진 질의응답(설치법 / 플러그인·스킬·MCP 구분 / API 토큰 / AI 에이전트 구축 /
> 수익화 / React·PHP 구현 / 유튜브 강의화)을 한 문서로 정리한 것입니다.

## 0. 저장소 주소

| 구분 | 주소 |
| --- | --- |
| 내 포크 (이 저장소) | https://github.com/bmshin94/openinterpreter |
| 작업 브랜치 | https://github.com/bmshin94/openinterpreter/tree/claude/loving-cori-l0cgs6 |
| 공식 업스트림 (Rust 신버전) | https://github.com/openinterpreter/openinterpreter |
| 원본 Python 구버전 (커뮤니티 유지) | https://github.com/endolith/open-interpreter |
| 상위 원조 프로젝트 (OpenAI Codex CLI) | https://github.com/openai/codex |
| 공식 문서 | https://www.openinterpreter.com/docs/terminal |
| Discord | https://discord.gg/Hvz9Axh84z |

관련 외부 프로젝트(하네스 참조 대상)

- Claude Code — https://docs.anthropic.com/en/docs/claude-code/overview
- Kimi Code — https://github.com/MoonshotAI/kimi-code
- Kimi CLI (레거시) — https://github.com/MoonshotAI/kimi-cli
- Qwen Code — https://github.com/QwenLM/qwen-code
- DeepSeek TUI — https://github.com/DeepSeek-TUI/DeepSeek-TUI
- SWE-agent — https://github.com/SWE-agent/SWE-agent
- agent-browser (웹 QA) — https://github.com/vercel-labs/agent-browser
- trycua / cua (데스크톱 QA) — https://github.com/trycua/cua
- Agent Client Protocol — https://agentclientprotocol.com/

---

## 1. 가장 먼저 짚어야 할 사실: CLAUDE.md와 실제 코드가 다릅니다

저장소 루트의 `CLAUDE.md`는 **구 Python 버전 Open Interpreter**("내 컴퓨터 자율 오퍼레이터",
자연어로 파일 정리·데이터 분석)를 설명하는 자동 생성 문서입니다.

하지만 실제 코드를 전수조사한 결과는 다릅니다.

| 항목 | CLAUDE.md 설명 | 실제 저장소 내용 |
| --- | --- | --- |
| 언어 | Python | **Rust 약 48.5만 줄** (`.rs` 3,924개) |
| 정체성 | 범용 컴퓨터 조작 비서 | **터미널 코딩 에이전트** |
| 기반 | 자체 구현 | **OpenAI Codex CLI 포크** |
| 실행 파일 | `interpreter` (Python 패키지) | `interpreter` / `i` (Rust 단일 바이너리) |

즉 이 저장소는 **"저가형 오픈소스 모델에 최적화된, OpenAI Codex CLI 기반 Rust 터미널
코딩 에이전트"** 입니다. `README.md` 하단에도 "This is the new Rust version of
Open Interpreter, based on Codex"라고 명시돼 있습니다.

### 실측 통계

```
총 파일          7,656개
Rust (.rs)       3,924개 / 485,305줄
스냅샷 테스트     944개
TypeScript        750개
Python            187개 (SDK 예제·스크립트)
Markdown          294개
Rust 크레이트     127개 (codex-rs/)
빌드 시스템       Bazel + Cargo + pnpm + Nix
라이선스          Apache-2.0
버전              0.0.43 / 0.0.44 (upstream Codex rust-v0.154.0 기준)
```

---

## 2. 폴더 구조 전수조사

| 경로 | 역할 |
| --- | --- |
| `codex-rs/` | **핵심.** Rust 크레이트 127개. 엔진 전체 |
| `codex-rs/core/` | 세션·턴 루프·설정·컨텍스트 관리 |
| `codex-rs/core/src/harness/` | **이 프로젝트의 핵심 차별점.** 하네스 에뮬레이션 |
| `codex-rs/tui/` | 터미널 UI (슬래시 명령, 승인 UI, 테마) |
| `codex-rs/cli/` | `interpreter` 명령어 엔트리포인트 |
| `codex-rs/exec/` | 비대화형 1회 실행(`interpreter exec`) |
| `codex-rs/model-provider-info/` | **호스팅 프로바이더 카탈로그 116개** |
| `codex-rs/sandboxing/`, `linux-sandbox/`, `windows-sandbox-rs/`, `bwrap/`, `mxc-sandbox/` | OS별 네이티브 샌드박스 |
| `codex-rs/execpolicy/` | 명령 실행 정책 엔진 |
| `codex-rs/skills/` | 스킬 로딩 + 번들 샘플 스킬 7종 |
| `codex-rs/codex-mcp/`, `mcp-server/`, `rmcp-client/` | MCP 클라이언트 + MCP 서버 모드 |
| `codex-rs/acp-server/` | Agent Client Protocol 서버 |
| `codex-rs/app-server/`, `app-server-daemon/` | 앱 서버(SDK/데스크톱 연결 지점) |
| `codex-rs/plugin/`, `core-plugins/` | 플러그인 시스템(실험적) |
| `codex-rs/hooks/` | 라이프사이클 훅 |
| `codex-rs/agent-roles/`, `agent-graph-store/` | 서브에이전트 역할/그래프 |
| `codex-rs/memories/` | 세션 간 기억(옵션, 기본 off) |
| `codex-rs/code-mode*`, `v8-poc/` | 코드 모드 / V8 런타임 실험 |
| `codex-rs/voice-host/`, `realtime-webrtc/` | 음성·실시간 실험 |
| `codex-rs/cloud-tasks*/` | 원격 태스크(실험적) |
| `codex-rs/secrets/`, `keyring-store/` | 자격증명 저장 |
| `codex-rs/otel/`, `analytics/`, `diagnostics/` | 관측·진단 |
| `codex-rs/product-info/` | **브랜딩 중앙화 지점** (포크 리브랜딩용) |
| `sdk/python`, `sdk/typescript` | Codex SDK 호환 예제/런타임 |
| `codex-cli/` | npm 패키지 래퍼 + 컨테이너 실행 스크립트 |
| `docs/` | 공식 문서 42편 (+ 중국어 번역 `docs/zh/`) |
| `docs-site/` | 문서 사이트 에셋 |
| `.codex/skills/` | 이 저장소 자체 개발용 스킬(코드리뷰, PR 베이비싯 등) |
| `scripts/install/` | `install.sh` / `install.ps1` 설치 스크립트 |
| `AGENTS.md` | 22KB 개발 규칙(에이전트가 읽는 기여 가이드) |
| `FORK_BRANDING.md` | 포크를 자기 브랜드로 바꾸는 공식 가이드 |
| `docs/open-interpreter-delta.md` | 업스트림 Codex와의 유지 차이 목록 |

---

## 3. 이게 뭐 하는 건가 — 한 줄 요약

**터미널에서 `i` 만 치면 들어오는, 내 저장소를 읽고 코드를 고치고 명령을 실행하는
AI 코딩 에이전트. 단, 비싼 모델에 묶이지 않도록 "싼 모델도 비싼 모델처럼
일하게 만드는" 하네스 전환 기능이 핵심.**

### 3-1. 핵심 차별점 ①: 하네스 에뮬레이션 (`/harness`)

같은 모델이라도 "어떤 시스템 프롬프트 / 어떤 도구 스키마 / 어떤 메시지 형식"으로
감싸느냐에 따라 성능이 크게 달라집니다. 이 프로젝트는 유명 에이전트 CLI들의
그 껍데기를 **Rust로 재구현**해서 스위치 하나로 바꿔 끼웁니다.

```
> /harness
native · claude-code · claude-code-bare · zcode · kimi-code
kimi-cli · qwen-code · deepseek-tui · swe-agent · minimal
```

실제 구현 파일 크기(= 재구현 깊이):

| 하네스 | 파일 | 크기 |
| --- | --- | --- |
| `claude-code` | `claude_code.rs` + 프롬프트 | 268KB + 36KB |
| `zcode` | `zcode.rs` + 도구 JSON | 118KB + 26KB |
| `kimi-cli` | `kimi_cli.rs` | 104KB |
| `terminus-2` | `terminus_2.rs` | 82KB |
| `opencode` | `opencode.rs` | 52KB |
| `kimi-code` | `kimi_code.rs` + 도구 JSON | 26KB + **82KB** |
| `swe-agent` / `mini-swe-agent` | 각각 | 33KB / 35KB |
| `deepseek-tui`, `qwen-code`, `minimal`, `pi`, `little-coder` | — | 9~33KB |

> 중요: 외부 CLI를 실제로 실행(shell out)하지 않습니다. **요청 형식만 똑같이 흉내내고,
> 도구 실행은 Open Interpreter 자체 Rust 런타임에서 돌립니다.**

프로바이더에 따라 하네스는 자동 선택됩니다.

| 감지된 모델 계열 | 자동 하네스 |
| --- | --- |
| Anthropic / Claude / `messages` API | `claude-code` |
| Kimi / Moonshot | `kimi-code` |
| Qwen / QwQ / DashScope | `qwen-code` |
| DeepSeek | `claude-code-bare` |

### 3-2. 핵심 차별점 ②: 프로바이더 116개

`codex-rs/model-provider-info/provider_catalog.json` 에 생성된 호스팅 프로바이더
**116개**가 들어 있습니다. (openrouter, groq, fireworks-ai, deepseek, moonshotai,
zai, zhipuai, minimax, nvidia, perplexity, google, github-copilot, huggingface,
poe, requesty, siliconflow, together류, 각종 중국·유럽 게이트웨이 등)

빌트인 런타임 프로바이더: `openai`, `amazon-bedrock`, `ollama`, `lmstudio`.

전송 방식(wire API) 3종:

| 값 | 전송 | 용도 |
| --- | --- | --- |
| `responses` | OpenAI Responses API | OpenAI, Bedrock, Ollama, LM Studio |
| `chat` | OpenAI 호환 Chat Completions | 대부분의 호스팅 프로바이더 |
| `messages` | Anthropic Messages | Anthropic 계열, ZCode |

### 3-3. 핵심 차별점 ③: 이식성(Portability) 철학

벤더 락인을 피하도록 **공용 표준 디렉터리**를 씁니다.

- 프로젝트 규칙: `AGENTS.md` (Claude의 `CLAUDE.md` 대응 공용 표준)
- 스킬: `.agents/skills/` (프로젝트) / `~/.agents/skills/` (개인)
- 제품 전용(`~/.openinterpreter/`)에는 설정·세션 상태만 보관
- 프로토콜: MCP, ACP, Codex exec 프로토콜

### 3-4. 안전장치: 샌드박스 + 승인 + 권한 프로파일

**샌드박스 모드 (기술적 경계)**

| 모드 | 동작 |
| --- | --- |
| `read-only` | 읽기만 |
| `workspace-write` | 작업 폴더 안에서만 쓰기, 네트워크 기본 차단 |
| `danger-full-access` | 경계 없음 (신뢰 환경 전용) |

**승인 정책 (언제 물어보나)**

| 정책 | 동작 |
| --- | --- |
| `untrusted` | 상태 변경 가능 행동 전에 항상 질문 |
| `on-request` | 샌드박스 안에서 돌리고, 권한 상승 때만 질문 |
| `never` | 안 물어봄 (샌드박스만이 방어선) |

**권한 프로파일 (신형, 더 세밀)** — 파일 glob 단위 read/write/deny + 도메인 단위
allow/deny까지 지정 가능. `.env` 거부, 특정 도메인만 허용 등이 가능합니다.

```toml
default_permissions = "project-edit"

[permissions.project-edit.filesystem.":workspace_roots"]
"." = "write"
"**/*.env" = "deny"

[permissions.project-edit.network.domains]
"**.github.com" = "allow"
"tracking.example.com" = "deny"
```

`--yolo` / `--dangerously-bypass-approvals-and-sandbox` 는 둘 다 끕니다. 반드시
일회용 VM/컨테이너에서만 쓰세요.

### 3-5. 번들 스킬 7종

| 스킬 | 역할 |
| --- | --- |
| `qa-testing` | **실제로 앱을 조작해서 검증.** 웹은 agent-browser, 데스크톱은 cua-driver |
| `skill-creator` | 스킬 만들기 도우미 |
| `skill-installer` | GitHub에서 스킬 설치 |
| `plugin-creator` | 플러그인 만들기 |
| `review-agent` | 코드 리뷰 에이전트 |
| `imagegen` | 이미지 생성 |
| `openai-docs` | 공식 문서 참조 |

### 3-6. 실행 모드 6가지

| 모드 | 명령 | 쓰임 |
| --- | --- | --- |
| 대화형 TUI | `i` / `interpreter` | 평소 작업 |
| 비대화형 | `interpreter exec "..."` | CI, 스크립트, 파이프라인 |
| MCP 서버 | `interpreter mcp-server` | 다른 에이전트가 날 도구로 호출 |
| ACP 에이전트 | `interpreter acp` | Zed 등 에디터 에이전트 패널 |
| 앱 서버 | `interpreter app-server` | SDK / 데스크톱 앱 연결 |
| 원격 | `interpreter --remote ws://...` | 원격 런타임 접속 |

### 3-7. 전체 CLI 서브커맨드 (코드에서 추출)

```
exec(e) review login logout mcp plugin mcp-server acp app-server
remote-control app completion update doctor sandbox debug
execpolicy apply(a) resume queue archive delete unarchive fork
cloud exec-server features agents
```

---

## 4. 쉽게 다시: 비유로 이해하기

**하네스 = 작업복**

같은 사람(모델)에게 어떤 작업복을 입히느냐로 결과가 달라집니다. 요리사 복장을
입히면 요리를 잘하고, 정비사 복장을 입히면 정비를 잘합니다. 이 프로젝트는
"클로드 코드 작업복", "Kimi 작업복", "Qwen 작업복"을 전부 만들어 두고 옷만
갈아입힙니다. **싼 모델에게 비싼 에이전트의 작업복을 입혀서 성능을 끌어올리는 것**이
전부입니다.

**프로바이더 = 전기 콘센트 어댑터 116개**

세계 어느 나라에 가도 꽂을 수 있는 멀티 어댑터. 어느 AI 회사든 꽂아서 씁니다.

**샌드박스 = 작업 반경 제한 목줄**

유능하지만 아직 못 믿는 신입에게 "이 폴더 밖으로는 나가지 마, 인터넷은 안 돼,
중요한 일은 나한테 물어봐"를 걸어두는 장치입니다.

**스킬 = 업무 매뉴얼 파일**

"릴리스 절차" 같은 반복 업무를 폴더(`SKILL.md`)로 적어두면, 필요할 때만
알아서 꺼내 읽습니다. 매번 프롬프트에 복붙할 필요가 없습니다.

**MCP = 외부 업체와의 전용 전화선**

Linear, DB, 사내 API 같은 외부 시스템에 셸 명령으로 더듬더듬 접근하는 대신
정식 연결선을 깔아줍니다.

**그래서 뭐가 다른가 (한 문장)**

> Claude Code / Codex / Cursor 는 "회사 모델에 묶인 코딩 에이전트"이고,
> 이건 "모델 회사를 내가 고르고 작업복까지 골라 끼우는 코딩 에이전트"입니다.

---

## 5. 질의응답

### Q1. 설치 및 사용법?

**설치 (macOS / Linux)**

```bash
curl -fsSL https://www.openinterpreter.com/install | sh
```

**설치 (Windows PowerShell)**

```powershell
irm https://www.openinterpreter.com/install.ps1 | iex
```

> ⚠️ 설치 스크립트를 파이프로 바로 실행하는 방식입니다. 보안상 먼저 내려받아
> 내용을 확인한 뒤 실행하는 것을 권합니다. (`curl -fsSL ... -o install.sh`)

**확인 → 시작**

```bash
interpreter --version
cd my-project
i                       # 또는 interpreter
```

**첫 실행 흐름**: 프로바이더 선택(ChatGPT 로그인 / API 키 / Ollama / LM Studio)
→ 모델 선택 → 작업 지시 → 승인 → 작업.

**자주 쓰는 명령**

| 작업 | 명령 |
| --- | --- |
| TUI 시작 | `i` |
| 프롬프트와 함께 시작 | `interpreter "explain this repo"` |
| 1회 실행 | `interpreter exec "summarize the current diff"` |
| 지난 세션 이어하기 | `interpreter resume --last` |
| 로컬 모델 사용 | `interpreter --oss "..."` |
| 읽기 전용 감사 | `interpreter --sandbox read-only "audit the auth flow"` |
| 설치 진단 | `interpreter doctor` |
| 업데이트 | `interpreter update` |

**주요 슬래시 명령**

`/model` `/harness` `/permissions` `/status` `/review` `/diff` `/init`
`/skills` `/mcp` `/plugins` `/agent` `/compact` `/fork` `/resume` `/hooks`
`/memories` `/ps` `/stop` `/fast` `/theme` `/mention` `/copy`

**설정 파일**: `~/.openinterpreter/config.toml`

```toml
model_provider = "moonshotai"
model = "kimi-k3"
harness = "kimi-code"
sandbox_mode = "workspace-write"
approval_policy = "on-request"
web_search = "cached"

[features]
hooks = true
multi_agent = true
memories = false
plugins = true
```

**제거**: `~/.openinterpreter/packages/standalone` 삭제 + `~/.local/bin` 심링크 제거.
사용자 데이터(`~/.openinterpreter`)는 남습니다.

---

### Q2. 이거 플러그인이야? 스킬이야? MCP야?

**셋 다 아닙니다. 이건 "호스트" 자체입니다.** 스킬·MCP·플러그인을 *소비하는* 쪽입니다.

| 질문 | 답 | 근거 |
| --- | --- | --- |
| 플러그인? | ❌ 아니다. 반대로 **플러그인을 받는 쪽** | `/plugins`, `interpreter plugin`, `codex-rs/plugin/` |
| 스킬? | ❌ 아니다. 반대로 **스킬을 읽는 쪽** | `.agents/skills/`, 번들 스킬 7종 |
| MCP? | 🔸 **양방향 모두 가능** | 클라이언트: `interpreter mcp add`. 서버: `interpreter mcp-server` |

정확한 분류:

```
Open Interpreter = 독립 실행형 터미널 코딩 에이전트 (CLI 앱)
  ├─ 안으로: 스킬 로드 · MCP 클라이언트 · 플러그인 · 훅 · 서브에이전트
  └─ 밖으로: MCP 서버 · ACP 에이전트 · 앱 서버 · exec 프로토콜
```

즉 **다른 에이전트의 부품이 될 수도 있고(MCP 서버 모드), 부품을 끼워 쓰는 본체가
될 수도 있는** 구조입니다. 비유하면 "확장 슬롯이 많은 데스크톱 본체"이고,
동시에 "다른 본체에 꽂히는 카드"도 될 수 있습니다.

---

### Q3. API 토큰을 사용해야 돼?

**아니요 — 세 가지 선택지가 있습니다.**

| 방식 | 토큰 필요? | 비용 | 설정 |
| --- | --- | --- | --- |
| **① 로컬 모델** | ❌ **전혀 불필요** | **0원** | Ollama / LM Studio 띄우고 `interpreter --oss` |
| ② 구독 로그인 | ❌ (브라우저 로그인) | 월 구독 | ChatGPT 로그인, Kimi For Coding 로그인 |
| ③ API 키 | ✅ 필요 | 종량제 | 환경변수 |

**①번이 이 프로젝트의 가장 큰 매력입니다.** 로컬 모델이면 토큰도, 요금도,
외부 전송도 없습니다. 빌트인 프로바이더에 `ollama`, `lmstudio`가 들어 있고
인증은 `none`입니다.

**API 키를 쓸 경우 환경변수**

```bash
export OPENAI_API_KEY=sk-...
export ANTHROPIC_API_KEY=...
export GEMINI_API_KEY=...
export DEEPSEEK_API_KEY=...
export KIMI_API_KEY=...        # 또는 MOONSHOT_API_KEY
export ZAI_API_KEY=...         # 또는 ZHIPU_API_KEY
```

**자격증명 저장 위치**

```toml
cli_auth_credentials_store = "auto"   # "auto" | "keyring" | "file"
```

`file` 이면 `~/.openinterpreter/auth.json` 에 저장됩니다. **비밀번호처럼 취급**하세요
(문서에도 명시). 가능하면 OS 키링(`keyring`)을 쓰세요.

**CI에서는** API 키가 사실상 필수입니다 (GitHub Actions secrets 사용).

---

### Q4. AI 에이전트를 구축하는 데 도움이 될까?

**매우 큽니다. 세 가지 층위로 쓸 수 있습니다.**

**① 바로 쓰는 엔진으로 (코딩 0줄)**

```python
# Python: Codex SDK + 바이너리만 교체
from codex_app_server import AppServerConfig, Codex

oi = AppServerConfig(launch_args_override=("interpreter","app-server","--listen","stdio://"))
with Codex(config=oi) as codex:
    thread = codex.thread_start(
        model_provider="moonshotai", model="kimi-k3",
        config={"harness": "kimi-code"},
    )
    print(thread.run("이 저장소 리뷰하고 첫 마이그레이션 단계 알려줘").final_response)
```

```ts
// TypeScript: 한 줄 변경
const codex = new Codex({ codexPathOverride: "interpreter" });
```

JSON 스트리밍도 바로 됩니다.

```bash
interpreter exec --json --output-schema schema.json "..." 
```

→ **에이전트 런타임을 직접 안 만들고도** 샌드박스·승인·도구·세션·재개가 전부 딸려옵니다.

**② 학습 교재로 (이게 사실 제일 가치 큼)**

| 배울 것 | 어디서 |
| --- | --- |
| 실전 에이전트 프롬프트 설계 | `harness/*_prompt.md` (claude_code 36KB, kimi_code 20KB, qwen 24KB) |
| 실전 도구 스키마 설계 | `kimi_code_tools.json` (82KB), `zcode_tools.json` (26KB) |
| 권한·샌드박스 아키텍처 | `sandboxing/`, `execpolicy/`, `permissions.md` |
| 멀티 프로바이더 추상화 | `model-provider-info/`, `chat-wire-compat/` |
| 서브에이전트 오케스트레이션 | `agent-roles/`, `agent-graph-store/` |
| 컨텍스트 압축 | `/compact`, `compact_remote_*` |
| 스킬/플러그인 설계 | `skills/`, `plugin/` |

**실제 상용 에이전트 하네스 6~10종의 프롬프트와 도구 정의가 평문으로 들어 있는
저장소**입니다. 이것만 읽어도 에이전트 설계 감각이 크게 올라갑니다.

**③ 포크해서 내 제품으로**

`FORK_BRANDING.md` 가 **공식 리브랜딩 가이드**입니다. `codex-rs/product-info/src/lib.rs`
한 곳만 바꾸면 제품명·명령어·설치 URL·릴리스 저장소가 전부 교체됩니다.
Apache-2.0이라 **상업적 사용·수정·재배포·상표 변경이 허용**됩니다.

**한계도 정직하게**

- Rust 48만 줄 + Bazel. 깊은 수정은 Rust 숙련도 필요
- 코딩/터미널 작업 특화. 범용 업무 자동화 에이전트로는 과한 선택
- `cloud`, `plugins`, `acp`, `voice` 등은 실험 단계
- 자체 SDK 없음 (OpenAI Codex SDK에 의존)

---

### Q5. 우리가 React나 PHP로 만들 수 있어?

**엔진을 React/PHP로 재작성 = 비현실적. 하지만 React/PHP로 "감싸는" 것은 매우 현실적이고, 그게 정답입니다.**

**왜 재작성이 비현실적인가**

| 이유 | 설명 |
| --- | --- |
| 규모 | Rust 48.5만 줄, 크레이트 127개 |
| OS 샌드박스 | seatbelt(macOS), bubblewrap/landlock(Linux), Windows 샌드박스 — JS/PHP로 불가 |
| 성능 | 터미널 TUI, 파일 워처, 스트리밍 |
| 프로세스 제어 | PTY, 프로세스 하드닝 |

**현실적인 방법: 래핑 (Open Interpreter를 백엔드 엔진으로)**

```
[React 프론트엔드]
      │ WebSocket / REST
      ▼
[Node.js 또는 PHP 백엔드]
      │ 프로세스 실행 + JSON 스트림 파싱
      ▼
interpreter exec --json  또는  interpreter app-server
      │
      ▼
[샌드박스 · 모델 프로바이더]
```

**Node.js(권장) 예시**

```js
import { spawn } from "node:child_process";

const p = spawn("interpreter", ["exec", "--json", "--sandbox", "workspace-write", prompt],
                { cwd: workdir });
p.stdout.on("data", chunk =>
  chunk.toString().split("\n").filter(Boolean)
    .forEach(line => ws.send(line))   // NDJSON 이벤트를 React로 그대로 중계
);
```

**PHP 예시**

```php
$cmd = 'interpreter exec --json --sandbox read-only ' . escapeshellarg($prompt);
$h = popen($cmd, 'r');
while (($line = fgets($h)) !== false) {
    echo "data: " . trim($line) . "\n\n";   // SSE로 스트리밍
    @ob_flush(); flush();
}
pclose($h);
```

**더 정식 경로 3가지**

1. **MCP 서버 모드** — `interpreter mcp-server`, 내 앱이 MCP 클라이언트
2. **ACP** — `interpreter acp`, 에디터형 UI를 만들 때
3. **앱 서버 + WebSocket** — `interpreter app-server --listen` + `--remote`

**PHP 쓸 때 보안 주의 (필수)**

- 사용자 입력을 반드시 `escapeshellarg()` 처리
- 절대 `--yolo` / `danger-full-access` 사용 금지
- `read-only` + 도메인 allowlist부터 시작
- 요청별 격리 컨테이너 + 타임아웃(`--timeout`) 필수
- 웹 서버 계정 권한 최소화

> 결론: **"React로 UI, PHP/Node로 중계, Rust 엔진은 그대로"** 가 정석입니다.
> 이게 바로 아래 수익화 아이디어의 기술적 토대입니다.

---

### Q6. 유튜브 강의 영상으로 제작 가능할까?

**가능성 매우 높습니다. 소재가 과잉일 정도로 많습니다.**

**강점**

- 시각적: 터미널에서 코드가 실시간으로 고쳐짐 → 썸네일/영상 임팩트 강함
- 후킹 포인트 명확: **"API 요금 0원으로 Claude Code 흉내내기"**
- 국내 한국어 콘텐츠 거의 없음 → 선점 가능
- 라이선스 Apache-2.0 → 영상/강의 제작·수익화에 법적 문제 없음
- 문서 42편이 사실상 커리큘럼 초안

**약점 / 주의**

- 버전 0.0.43~0.0.44 → 빠르게 변함. "버전 명시 + 날짜 고정" 필수
- 영문 터미널 중심 → 초보 진입장벽. 자막·확대·폰트 크게 필요
- 실험 기능 다수 → "실험적"임을 명확히 고지
- 설치가 `curl | sh` → 보안 경고 반드시 포함
- `--yolo` 시연은 반드시 VM/컨테이너에서 + 경고 자막

**추천 시리즈 커리큘럼 (12편)**

| # | 제목 | 길이 | 핵심 |
| --- | --- | --- | --- |
| 1 | API 요금 0원으로 AI 코딩 에이전트 쓰기 | 10분 | 설치 → 첫 작업 (훅킹용 1편) |
| 2 | Ollama + Open Interpreter 완전 로컬 세팅 | 15분 | 토큰 없이 돌리기 |
| 3 | `/harness` 의 정체 — 싼 모델을 비싸게 쓰는 법 | 12분 | **최대 차별 포인트** |
| 4 | 프로바이더 116개 전부 뜯어보기 | 15분 | 가성비 비교 |
| 5 | AI에게 내 컴퓨터를 안전하게 맡기는 법 | 15분 | 샌드박스·권한·승인 |
| 6 | AGENTS.md와 스킬로 내 작업 자동화 | 18분 | 실전 생산성 |
| 7 | MCP 연결해서 Notion·DB·Linear 쓰기 | 15분 | 통합 |
| 8 | GitHub Actions에 AI 코드리뷰 붙이기 | 15분 | `interpreter exec` CI |
| 9 | React + Node로 웹 UI 만들기 | 25분 | 래핑 실습 |
| 10 | 상용 에이전트 프롬프트 해부 | 20분 | 하네스 프롬프트 읽기(고급) |
| 11 | 포크해서 내 브랜드 CLI 만들기 | 20분 | `FORK_BRANDING.md` |
| 12 | 에이전트에게 QA까지 맡기기 | 15분 | `qa-testing` 스킬 |

**제작 팁**

- 1편은 "결과 먼저" — 첫 15초에 실제로 코드가 고쳐지는 장면
- 전편 동일 샘플 프로젝트 사용 → 재생목록 체류 시간 ↑
- 로컬 모델 편과 클라우드 편을 분리 → 각각 다른 타겟
- 설명란에 `config.toml` 전문 + GitHub 저장소 링크 고정
- "업데이트 로그" 고정 댓글로 버전 변경 대응

---

## 6. 수익화 아이디어 (상세)

**전제**: 라이선스 Apache-2.0 → 상업적 사용·수정·재배포 가능. 단
(1) LICENSE·NOTICE 유지, (2) 변경 사실 명시, (3) **"Open Interpreter" 이름으로
오해를 유발하지 말 것** (상표 아님 ≠ 혼동 유발 허용). `FORK_BRANDING.md` 가
독자 브랜드를 만드는 공식 경로를 제공합니다.

### 티어 1 — 즉시 가능 (자본 거의 0, 1~4주)

| # | 아이디어 | 수익 모델 | 예상 | 난이도 |
| --- | --- | --- | --- | --- |
| 1 | **한국어 강의·전자책** | 인프런/클래스101/유데미, PDF | 편당 3~15만원, 강의 100~500만원 | ★☆☆ |
| 2 | **유튜브 채널** | 광고 + 멤버십 + 협찬 | 구독 1만 기준 월 50~300만원 | ★☆☆ |
| 3 | **유료 스킬 팩 판매** | Gumroad/Lemon Squeezy 번들 | 팩당 2~5만원 | ★☆☆ |
| 4 | **도입 컨설팅 / 1:1 세팅** | 시간제 | 시간당 10~30만원 | ★☆☆ |
| 5 | **뉴스레터 (에이전트 생태계)** | 유료 구독 | 월 1만원 × 구독자 | ★☆☆ |

→ **①+②를 묶는 것이 최적 진입점.** 유튜브로 신뢰 쌓고 강의로 전환.

### 티어 2 — 중기 (1~3개월, 개발 필요)

| # | 아이디어 | 설명 | 수익 모델 |
| --- | --- | --- | --- |
| 6 | **"AI 코드리뷰 봇" SaaS** | `interpreter exec --sandbox read-only` + GitHub App. 싼 모델로 리뷰 → 원가 극저 | 저장소당 월 1~3만원 |
| 7 | **브랜디드 사내 CLI 구축** | `FORK_BRANDING.md` 로 리브랜딩 + 사내 MCP·스킬·권한 프로파일 세팅 | 구축 500~3000만원 + 연 유지보수 |
| 8 | **업종별 스킬 마켓플레이스** | 법률/회계/의료/커머스 업무 스킬 팩 + 유통 | 판매 수수료 20~30% |
| 9 | **React 웹 UI 상품화** | Q5의 래핑을 제품으로. 비개발자용 GUI | 셀프호스팅 라이선스 or SaaS |
| 10 | **로컬 AI 코딩 머신 세팅 서비스** | 하드웨어 추천 + Ollama + OI 튜닝 + 교육 | 건당 100~500만원 (하드웨어 별도) |

### 티어 3 — 장기 / 고수익 (3개월+)

| # | 아이디어 | 왜 이 프로젝트가 유리한가 |
| --- | --- | --- |
| 11 | **보안·규제 산업용 온프레미스 에이전트** | 금융·공공·의료는 "코드가 외부로 안 나감"이 절대 요구. **로컬 모델 + 샌드박스 + 도메인 allowlist + 감사 로그(otel)** 가 이미 구현돼 있음. 가장 비싼 시장 |
| 12 | **모델 비용 최적화 플랫폼** | 하네스 × 116 프로바이더 조합으로 "같은 품질 최저가 경로"를 찾아 라우팅. 절감액의 20~30% 과금 |
| 13 | **에이전트 벤치마크·평가 서비스** | 하네스 스위치가 있어 **동일 조건 비교 실험**이 가능. 프로바이더에게 리포트 판매, 또는 공개 리더보드로 트래픽 확보 |
| 14 | **레거시 마이그레이션 자동화** | PHP5→8, jQuery→React 등. 저가 모델 대량 병렬 + 서브에이전트 | 프로젝트당 수천만원 |
| 15 | **에이전트 전용 호스팅** | `app-server` + `--remote` 로 팀 공유 에이전트 호스팅 | 시트당 월 과금 |

### 수익화 전략 추천 로드맵

```
1개월차   유튜브 1~3편 + 블로그   →  신뢰 자산 구축 (수익 거의 0)
2~3개월차 강의/전자책 출시        →  첫 현금화, 월 100~500만원
3~4개월차 컨설팅 유입 시작         →  유튜브가 리드 생성기 역할
4~6개월차 코드리뷰 봇 MVP 또는     →  반복 매출(MRR) 확보
          사내 CLI 구축 1건
6개월차+  규제산업 온프레미스로 확장 →  고단가 B2B
```

### 왜 이게 통할 가능성이 높은가

1. **비용 통증이 실재** — Claude Code/Cursor 구독비 + API 요금 부담이 크다
2. **이 프로젝트가 정확히 그 통증을 겨냥** — "저가 모델 최적화"가 공식 포지셔닝
3. **데이터 주권 수요** — 코드 외부 유출 금지 기업이 많고, 로컬 모델이 이미 지원
4. **한국어 콘텐츠 공백** — 선점 여지
5. **Apache-2.0** — 법적 리스크 없이 상업화 가능

### 리스크 (반드시 인지)

| 리스크 | 완화 |
| --- | --- |
| 업스트림(Codex) 변화에 종속 | 포크 유지 비용 계상, delta 문서 참고 |
| 버전 0.0.x — 아직 초기 | 특정 버전 고정(pin)해서 제품화 |
| OpenAI/Anthropic이 유사 기능 공식 출시 | 차별점을 "온프레미스·규제·비용최적화"로 |
| `curl \| sh` 설치 + 샌드박스 오설정 | 고객 환경에서는 컨테이너 격리 필수 |
| 강의 내용 빠른 노후화 | 버전·날짜 명시 + 업데이트 영상 |
| 모델 품질 변동 | 벤치마크로 상시 검증 |

### 가장 추천하는 단일 조합

> **"유튜브 시리즈(무료) → 한국어 종합 강의(유료) → 기업 온프레미스 구축 컨설팅"**
>
> 자본이 거의 안 들고, 앞 단계가 뒤 단계의 영업 채널이 되며,
> 마지막 단계가 단가가 가장 높습니다.

---

## 7. 최종 요약

| 질문 | 답 |
| --- | --- |
| 이게 뭐야? | OpenAI Codex CLI를 포크한 **Rust 터미널 코딩 에이전트** (CLAUDE.md의 Python 설명은 구버전) |
| 가장 큰 특징? | **하네스 에뮬레이션** — 싼 모델에 비싼 에이전트의 프롬프트/도구를 입힘 |
| 플러그인/스킬/MCP? | 전부 아님. **이것들을 쓰는 호스트 앱**. 단 MCP는 서버로도 동작 |
| 토큰 필요? | **로컬 모델이면 불필요·무료.** 클라우드면 구독 또는 API 키 |
| 에이전트 구축에 도움? | 매우 큼. 엔진으로 즉시 사용 + 상용 하네스 프롬프트 교재 + 포크해 제품화 |
| React/PHP로 가능? | 재작성은 비현실적. **래핑(React UI + Node/PHP 중계 + Rust 엔진)이 정답** |
| 유튜브 가능? | 매우 적합. 12편 커리큘럼 제안. 버전 명시·보안 경고 필수 |
| 수익화? | 티어1 콘텐츠/컨설팅 → 티어2 SaaS/사내구축 → 티어3 규제산업 온프레미스 |
| 라이선스 | **Apache-2.0** (상업적 사용 가능, NOTICE 유지 + 변경 명시 + 브랜드 혼동 금지) |

---

*분석 기준일: 2026-10-07 · 기준 커밋: `26774f5c` · 버전 0.0.43/0.0.44 (upstream Codex rust-v0.154.0)*
