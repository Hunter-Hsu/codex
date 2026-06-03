# Codex Codebase Guide — 架構概覽與快速入門

> 適用對象：新加入的維護者或貢獻者，目標是在一個工作天內對整個 codebase 有足夠的地圖感。

---

## 目錄

1. [什麼是 Codex](#1-什麼是-codex)
2. [Repo 頂層結構](#2-repo-頂層結構)
3. [技術棧一覽](#3-技術棧一覽)
4. [本機開發環境建置](#4-本機開發環境建置)
5. [Codex-RS：Rust 主引擎](#5-codex-rs-rust-主引擎)
   - [Crate 分層地圖](#crate-分層地圖)
   - [核心 crate 詳解](#核心-crate-詳解)
6. [完整資料流：一條使用者訊息的生命週期](#6-完整資料流一條使用者訊息的生命週期)
7. [Agent 迴圈：Turn 執行邏輯](#7-agent-迴圈turn-執行邏輯)
8. [Model API 整合](#8-model-api-整合)
9. [Tool 執行與沙箱機制](#9-tool-執行與沙箱機制)
10. [Plugin / MCP / Skills 系統](#10-plugin--mcp--skills-系統)
11. [設定檔系統](#11-設定檔系統)
12. [狀態持久化](#12-狀態持久化)
13. [TUI 架構](#13-tui-架構)
14. [Codex-CLI：Node.js 包裝層](#14-codex-cli-nodejs-包裝層)
15. [SDK（TypeScript / Python）](#15-sdktypescript--python)
16. [常用開發指令](#16-常用開發指令)
17. [新功能開發路徑建議](#17-新功能開發路徑建議)
18. [關鍵檔案速查表](#18-關鍵檔案速查表)

---

## 1. 什麼是 Codex

Codex 是 OpenAI 推出的 **本機 AI 程式碼代理（coding agent）**，可以在使用者的電腦上直接執行 shell 指令、讀寫檔案、呼叫 API，同時透過作業系統級沙箱確保安全性。

核心特色：
- **多模型支援**：OpenAI、OpenRouter、LMStudio（本機）、Ollama（本機）
- **多平台**：macOS、Linux、Windows（透過 WSL2）
- **可擴充**：MCP（Model Context Protocol）讓 Codex 能連接外部工具伺服器；Skills 允許使用者定義自訂指令
- **可稽核**：每次對話都完整記錄為 JSONL 事件日誌，支援 `resume` / `fork`

---

## 2. Repo 頂層結構

```
codex/
├── codex-cli/          # Node.js 薄包裝層，負責分發與啟動 Rust binary
├── codex-rs/           # 主引擎：Rust monorepo，113+ crates
├── sdk/
│   ├── typescript/     # TypeScript SDK (@openai/codex-sdk)
│   └── python/         # Python SDK (openai-codex)
├── docs/               # 使用者與開發者文件
├── scripts/            # 建置與工具腳本
├── tools/              # 開發工具（linter helpers 等）
├── patches/            # Cargo 依賴補丁
├── third_party/        # 第三方依賴
├── .codex/             # 此 repo 自身的 Codex skills / 設定
├── .github/            # CI/CD workflows
├── justfile            # 常用指令捷徑（just fmt, just test…）
├── .bazelrc            # Bazel 建置設定（可重現的 hermetic build）
└── .bazelversion       # 釘定 Bazel 版本
```

**最重要的子目錄是 `codex-rs/`**，所有核心邏輯都在這裡。

---

## 3. 技術棧一覽

| 層級 | 技術 |
|------|------|
| 核心引擎 | Rust（Tokio async runtime） |
| 終端 UI | Ratatui + Crossterm |
| CLI 解析 | Clap |
| HTTP / WebSocket | Reqwest + tokio-tungstenite |
| 序列化 | Serde / serde_json |
| 資料庫 | SQLite via sqlx |
| HTTP 框架（app-server） | Axum |
| Tracing / 日誌 | tracing crate + RUST_LOG |
| MCP | rmcp（Rust MCP SDK） |
| 建置 | Cargo + Bazel（雙軌） |
| 分發包裝 | Node.js（codex-cli） |

---

## 4. 本機開發環境建置

```bash
# 1. Clone
git clone https://github.com/openai/codex.git
cd codex/codex-rs

# 2. 安裝 Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustup component add rustfmt clippy

# 3. 安裝工具
cargo install --locked just
cargo install --locked cargo-nextest   # 可選，用於 just test

# 4. 建置
cargo build

# 5. 啟動（互動 TUI 模式）
cargo run --bin codex -- "解釋這個 codebase 的架構"

# 6. 非互動模式
cargo run --bin codex -- exec "產生一個測試檔"
```

### 查看即時日誌

```bash
# TUI 模式的日誌寫入此檔
tail -F ~/.codex/log/codex-tui.log

# 增加 log 詳細度
RUST_LOG=codex_core=debug,codex_tui=debug cargo run --bin codex -- "prompt"
```

---

## 5. Codex-RS：Rust 主引擎

`codex-rs/` 是一個 Cargo workspace，包含 113+ 個 crates，以功能分層。

### Crate 分層地圖

```
┌─────────────────────────────────────────────────────┐
│  CLI / App Entry Point                              │
│  cli · app-server · app-server-daemon               │
├─────────────────────────────────────────────────────┤
│  Presentation / UI                                  │
│  tui · custom_terminal                              │
├─────────────────────────────────────────────────────┤
│  Core Agent Engine                                  │
│  core (session · turn · client · tools)             │
│  codex-thread · thread-store · state · rollout      │
├──────────────────────┬──────────────────────────────┤
│  Tool Execution      │  Model / Provider            │
│  tools · exec        │  model-provider              │
│  exec-server         │  model-provider-info         │
│  execpolicy          │  models-manager              │
│  sandboxing          │  chatgpt · login             │
│  bwrap · linux-sandbox                              │
│  windows-sandbox-rs  │                             │
├──────────────────────┴──────────────────────────────┤
│  Plugin / MCP / Skills                              │
│  codex-mcp · mcp-server · rmcp-client              │
│  plugin · core-plugins · connectors · skills       │
├─────────────────────────────────────────────────────┤
│  Configuration & Infrastructure                     │
│  config · features · hooks · otel                   │
│  agent-identity · agent-graph-store                 │
│  protocol · async-utils · utils/*                   │
└─────────────────────────────────────────────────────┘
```

### 核心 crate 詳解

#### `cli` — 程式進入點
**路徑：** `codex-rs/cli/src/main.rs`

Clap-based CLI，解析命令列後分發到各模式：

```
codex                   → 預設進入互動 TUI
codex exec "..."        → 非互動執行
codex review <file>     → Code review 模式
codex mcp-server        → 作為 MCP server 啟動
codex app-server        → 啟動 HTTP/WebSocket 伺服器
codex resume --last     → 恢復上次 session
codex fork              → 從某 session 分叉
codex login / logout    → 認證管理
```

#### `core` — Agent 引擎核心
**路徑：** `codex-rs/core/src/`

這是整個系統最重要的 crate，包含：
- `session/mod.rs` — Session 狀態機（136KB，系統核心）
- `session/turn.rs` — 單次對話回合的執行邏輯
- `session/session.rs` — Session 結構定義
- `client.rs` — Model API 呼叫（WebSocket streaming）
- `codex_thread.rs` — Thread/Conversation 抽象層

#### `protocol` — 事件協議定義
所有跨組件的訊息格式在此定義，新開發者應先熟悉 `Event` enum 的結構。

#### `tools` — Tool 執行管線
負責 tool 的分派與執行協調，`orchestrator.rs` 是主入口。

---

## 6. 完整資料流：一條使用者訊息的生命週期

```
使用者輸入: "幫我寫這個函式的單元測試"
    │
    ▼
① codex-cli/bin/codex.js
   偵測平台（macOS/Linux/Windows ARM64/x64）
   → 找到對應的 Rust binary
   → spawn child process，轉發 SIGINT/SIGTERM
    │
    ▼
② cli/src/main.rs
   Clap 解析命令列參數
   沒有 subcommand → 進入 TUI 模式
   有 `exec` subcommand → 非互動模式
    │
    ▼
③ Session 初始化 (core/src/session/mod.rs)
   讀取 ~/.codex/config.toml + .codex/config.toml
   認證（OAuth token 或 API key，from keyring）
   建立 ModelClient（選定 provider）
   初始化 MCP ConnectionManager
   載入 plugins / skills
    │
    ▼
④ Turn 準備 (core/src/session/turn.rs)
   讀取 git 狀態、最近檔案變更
   收集對話歷史（必要時 compact）
   列出可用 tools（built-in + MCP tools）
   解析 skill mentions
    │
    ▼
⑤ Prompt 建構
   系統指令 + 使用者 instructions
   對話歷史
   可用 tools schema（JSON Schema 格式）
   使用者訊息
    │
    ▼
⑥ Model API 呼叫 (core/src/client.rs)
   建立 ModelClientSession（per-turn WebSocket）
   可選：prewarm connection（response.create, generate=false）
   送出 Responses API 請求
   WebSocket 串流接收事件
    │
    ▼
⑦ 解析串流回應 (core/src/session/turn.rs)
   ┌──────────────────────────────────┐
   │ ContentDelta → 累積文字          │
   │ ToolCall → 提取 name + arguments │
   │ MessageComplete → 結束           │
   └──────────────────────────────────┘
    │           │
    │           ▼（若有 tool call）
    │  ⑧ Tool 執行 (tools/orchestrator.rs)
    │     ToolRouter 找到對應 handler
    │     檢查 approval policy（需要使用者確認？）
    │     套用 sandbox policy
    │     在沙箱中執行命令
    │     捕捉 stdout / stderr / exit code
    │     格式化為 ToolOutput
    │     → 附加到 prompt，回到步驟 ⑥ 繼續迴圈
    │
    ▼（最終文字回應）
⑨ 結束 Turn
   記錄到對話歷史
   持久化至 SQLite + JSONL 事件日誌
   回傳給 TUI / CLI 顯示
```

---

## 7. Agent 迴圈：Turn 執行邏輯

**關鍵檔案：** `codex-rs/core/src/session/turn.rs`

Codex 使用**多輪 tool-use 迴圈**，直到模型回傳最終文字回應為止：

```
run_turn(user_input) {
    1. pre_sampling_compact()     // 超過 token limit 時壓縮歷史
    2. record_context_updates()   // 更新 git 狀態、檔案快照
    3. build_prompt()             // 組合完整 prompt
    
    loop {
        4. stream_responses()     // 呼叫模型 API，串流事件
        
        for each event {
            if ToolCall {
                5. execute_tool()  // 執行 tool
                append_tool_result_to_prompt()
                continue loop      // ← 繼續向模型送結果
            }
            if FinalText {
                return text        // ← 跳出迴圈
            }
        }
    }
}
```

**Session 結構（`session.rs`）：**

```rust
struct Session {
    conversation_id: ThreadId,       // 唯一識別此對話
    tx_event: Sender<Event>,         // 向 TUI 廣播事件
    agent_status: watch::Sender<AgentStatus>,
    state: Mutex<SessionState>,      // 當前 session 狀態
    active_turn: Mutex<Option<ActiveTurn>>,
    mailbox: Mailbox,                // 接收使用者訊息
    services: SessionServices,       // ModelClient, MCP, 等共享服務
}
```

---

## 8. Model API 整合

**關鍵檔案：** `codex-rs/core/src/client.rs`

### 兩層 Client 設計

| 類別 | 生命週期 | 職責 |
|------|---------|------|
| `ModelClient` | Session 級（長存） | 認證、provider 選擇 |
| `ModelClientSession` | Turn 級（per-turn） | WebSocket 連線、請求串流 |

### 請求流程

```
1. 讀取認證（keyring OAuth token 或 OPENAI_API_KEY）
2. 選擇 provider（OpenAI / OpenRouter / LMStudio / Ollama）
3. 可選 prewarm：response.create（generate=false）預熱 WebSocket
4. 透過 WebSocket 送出 Responses API 請求
5. 串流解析 SSE 事件：ContentDelta / ToolCall / Reasoning / Done
6. 若 WebSocket 失敗，fallback 到 HTTP SSE
```

### 支援 Provider

- `openai` — OpenAI 正式 API（預設）
- `openrouter` — 多模型代理
- `lmstudio` — 本機模型（OpenAI 相容介面）
- `ollama` — 本機模型

設定於 `~/.codex/config.toml` 的 `[model]` 區段。

---

## 9. Tool 執行與沙箱機制

**關鍵 crates：** `tools` · `exec` · `sandboxing` · `execpolicy`

### Tool 種類

| 種類 | 說明 |
|------|------|
| Built-in Tools | 檔案讀寫、shell 執行、搜尋等核心工具 |
| MCP Tools | 來自外部 MCP server 的工具 |
| Skills | 使用者定義的腳本工具（.codex/skills/） |
| Dynamic Tools | 根據上下文在執行期動態生成 |

### 沙箱機制（依平台）

| 平台 | 技術 |
|------|------|
| Linux | Bubblewrap（bwrap）+ Landlock |
| macOS | Seatbelt（sandbox-exec） |
| Windows | MSFT Restricted Token |

### Approval Policy

執行工具前，系統會依 config 設定決定是否需要使用者確認：
- `permit` — 全部允許，不詢問
- `request` — 高風險操作需使用者確認（預設）
- `deny` — 全部拒絕

---

## 10. Plugin / MCP / Skills 系統

### MCP（Model Context Protocol）

Codex 同時扮演兩個角色：
- **MCP Client** — 連接外部 MCP server（`codex-mcp` crate），取得額外工具
- **MCP Server** — 自身暴露為 MCP server（`codex mcp-server`），供其他工具呼叫

```
外部 MCP Server ←→ codex-mcp ←→ Core Agent
（提供 GitHub, Slack, DB 等工具）
```

### Skills 系統

Skills 是儲存在 `.codex/skills/` 的使用者定義指令：

```
.codex/skills/my-workflow/
├── SKILL.md        # Prompt 模板，被注入到 model context
├── agents/         # 子 agent 指令
├── scripts/        # 輔助腳本
└── references/     # 參考文件
```

使用者可在訊息中提及 skill 名稱（`@my-workflow`），或設定自動注入。

---

## 11. 設定檔系統

**關鍵 crate：** `config`

### 設定檔載入優先順序（後者覆蓋前者）

```
~/.codex/config.toml              # 全域使用者設定
    +
.codex/config.toml                # 專案級設定（git repo 根目錄）
    +
~/.codex/profiles/v2/<name>/config.toml  # 具名 profile（選用）
    +
CLI 參數 -c key=value             # 執行期覆蓋
```

### 重要設定項

```toml
[model]
provider = "openai"
name = "gpt-4o"

[approval]
policy = "request"   # permit | request | deny

[sandbox]
mode = "bubblewrap"  # 依平台：bubblewrap | seatbelt | restricted_token

[hooks]
# 擴充行為的 hook 腳本

[tools]
# 工具設定

[plugins]
# 啟用的 plugins / connectors
```

---

## 12. 狀態持久化

**關鍵 crates：** `state` · `rollout` · `thread-store`

### 三層持久化

| 層 | 儲存媒介 | 用途 |
|----|---------|------|
| In-Memory | Session 物件 | 當前 turn 的即時狀態 |
| SQLite | `~/.codex/state.db` | Session 索引、設定快照 |
| JSONL Event Log | `~/.codex/rollout/<date>/<uuid>/events.jsonl` | 完整事件歷史，支援 replay |

### Session Resume / Fork

```bash
codex resume --last          # 恢復上次 session
codex resume <session-uuid>  # 恢復指定 session
codex fork                   # 從當前 session 分叉新 session
```

---

## 13. TUI 架構

**關鍵 crate：** `tui`，基於 [Ratatui](https://ratatui.rs/) + Crossterm

主要組件：

| 組件 | 職責 |
|------|------|
| `App` | 主應用狀態與事件迴圈 |
| `ChatWidget` | 對話訊息顯示 |
| `ExecCell` | Shell 執行輸出顯示 |
| `HistoryCell` | 對話歷史導覽 |
| `Keymap` | 鍵盤快捷鍵處理 |

TUI 透過 `tx_event: Sender<Event>` channel 接收 core 的非同步事件，實現即時串流顯示。

---

## 14. Codex-CLI：Node.js 包裝層

**關鍵檔案：** `codex-cli/bin/codex.js`（231 行）

這個薄包裝層的唯一職責是**平台偵測 + 啟動 Rust binary**：

```javascript
// 1. 偵測平台
const platform = process.platform  // darwin / linux / win32
const arch = process.arch           // x64 / arm64

// 2. 對應平台包
// @openai/codex-linux-x64
// @openai/codex-darwin-arm64
// @openai/codex-win32-x64  ...

// 3. spawn native binary，轉發 stdin/stdout/stderr
// 設定 CODEX_MANAGED_BY_NPM, CODEX_MANAGED_PACKAGE_ROOT 環境變數
```

**為何這樣設計？** npm/bun 生態系可以用 optional dependencies 機制，依平台只下載對應的 pre-built binary，避免使用者需要自行編譯 Rust。

---

## 15. SDK（TypeScript / Python）

### TypeScript SDK

**路徑：** `sdk/typescript/`  
**Package：** `@openai/codex-sdk`

提供程式化呼叫 Codex agent 的介面，適合整合到 CI/CD 或自動化工作流。

### Python SDK

**路徑：** `sdk/python/`  
**Package：** `openai-codex`

Python 版本的程式化介面。

**範例參考：**
- `sdk/typescript/samples/`
- `sdk/python/examples/`

---

## 16. 常用開發指令

```bash
# 格式化所有 Rust 程式碼
just fmt

# Clippy 檢查並自動修正（指定 crate）
just fix -p codex-core

# 執行測試（全部）
just test

# 執行特定 crate 測試
cargo test -p codex-tui

# 建置並執行（互動模式）
just codex "你的 prompt"

# 非互動模式執行
just exec "shell command"

# 完整 Clippy 檢查
just clippy

# 查看 session 日誌
tail -F ~/.codex/log/codex-tui.log
```

---

## 17. 新功能開發路徑建議

根據你要改動的面向，從以下位置切入：

| 想改的功能 | 從這裡開始 |
|----------|-----------|
| TUI 介面 / 顯示邏輯 | `codex-rs/tui/src/` |
| Agent 核心邏輯（prompt 建構、turn 流程） | `codex-rs/core/src/session/` |
| Model API 呼叫（provider、streaming） | `codex-rs/core/src/client.rs` |
| Tool 執行 / 沙箱 | `codex-rs/tools/src/` + `codex-rs/core/src/tools/handlers/` |
| 設定檔解析與驗證 | `codex-rs/config/src/` |
| MCP / Plugin 整合 | `codex-rs/codex-mcp/src/` |
| Skills 系統 | `codex-rs/skills/src/` |
| CLI 命令新增 | `codex-rs/cli/src/main.rs` |
| HTTP API（app-server） | `codex-rs/app-server/src/` |
| TypeScript SDK | `sdk/typescript/src/` |
| Python SDK | `sdk/python/src/openai_codex/` |

### 開發流程

```
1. 確認影響範圍：找到對應 crate
2. 寫測試（先讓測試失敗）
3. 實作功能
4. just fmt && just fix -p <crate>
5. cargo test -p <crate>（或 just test）
6. 在本機執行 just codex "..." 驗證
7. 提交前確認 CI 相關 check 都過
```

---

## 18. 關鍵檔案速查表

| 檔案 | 說明 |
|------|------|
| `codex-cli/bin/codex.js` | Node.js 入口，平台偵測 + 啟動 Rust binary |
| `codex-rs/cli/src/main.rs` | CLI 命令解析，系統入口 |
| `codex-rs/core/src/session/mod.rs` | Session 狀態機（最核心，136KB） |
| `codex-rs/core/src/session/turn.rs` | Turn 執行迴圈（agent loop） |
| `codex-rs/core/src/session/session.rs` | Session 結構定義 |
| `codex-rs/core/src/client.rs` | Model API client（WebSocket streaming） |
| `codex-rs/core/src/codex_thread.rs` | Thread/Conversation 抽象 |
| `codex-rs/tools/src/lib.rs` | Tool 執行管線入口 |
| `codex-rs/protocol/src/` | 跨組件事件協議定義 |
| `codex-rs/config/src/config_toml.rs` | 設定檔結構定義 |
| `codex-rs/tui/src/lib.rs` | TUI 應用入口 |
| `codex-rs/app-server/src/lib.rs` | App server（HTTP/WebSocket） |
| `codex-rs/Cargo.toml` | Workspace 定義，所有 crate 列表 |
| `justfile` | 常用指令捷徑 |
| `docs/install.md` | 完整安裝與建置說明 |
| `docs/contributing.md` | 貢獻流程說明 |

---

*文件最後更新：2026-05-27。如發現過時資訊，請直接修改此檔或在 issue 中反映。*
