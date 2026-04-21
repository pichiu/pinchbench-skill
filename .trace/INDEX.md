# PinchBench — 專案總覽與速查

## 一句話總結

PinchBench 是一套 CLI benchmark 工具，透過向 OpenClaw agent 發送真實世界任務（行事曆、Email、程式碼等），衡量各 LLM 模型作為 agent 大腦的實際表現，並將結果上傳至公開排行榜 [pinchbench.com](https://pinchbench.com)。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | ≥3.10 | 主要執行環境 |
| Package manager | uv | latest | 依賴管理與腳本執行 |
| Build system | setuptools-scm | ≥8 | 打包與版本管理 |
| Config parser | PyYAML | ≥6.0.1 | 任務 frontmatter 解析 |
| 外部 CLI | openclaw | runtime | Agent 管理與任務執行 |
| 外部 CLI | fws (npm) | optional | Google Workspace mock server |
| 外部 CLI | gws | optional | Google Workspace CLI |
| 外部 API | OpenRouter | - | LLM 模型路由（主要） |
| 外部 API | Anthropic API | - | 直接 judge 模式 |
| 外部 API | OpenAI API | - | 直接 judge 模式 |
| 外部 API | api.pinchbench.com | - | 結果上傳與排行榜 |
| Linting | ruff | 0.15.6 | 程式碼 lint |
| Testing | pytest | ≥7.4.0 | 單元測試 |
| CI/CD | GitHub Actions | - | lint / release / 任務計數更新 |

---

## 關鍵指令速查

```bash
# 安裝依賴（使用 uv）
uv sync

# 執行 benchmark（全部任務）
./scripts/run.sh --model openrouter/anthropic/claude-sonnet-4

# 只跑自動評分任務（較快）
./scripts/run.sh --model openrouter/anthropic/claude-sonnet-4 --suite automated-only

# 跑指定任務
./scripts/run.sh --model openrouter/openai/gpt-4o --suite task_calendar,task_stock

# 不上傳結果
./scripts/run.sh --model openrouter/anthropic/claude-sonnet-4 --no-upload

# 多次執行取平均
./scripts/run.sh --model openrouter/anthropic/claude-sonnet-4 --runs 3

# 使用直接 API 作為 judge（繞過 OpenClaw）
./scripts/run.sh --model openrouter/openai/gpt-4o --judge anthropic/claude-sonnet-4-5-20250514

# 使用自訂 endpoint（如 ollama）
./scripts/run.sh --model my-local-model --base-url http://localhost:11434/v1

# 申請 API token（一次性）
./scripts/run.sh --register

# 上傳既有結果
./scripts/run.sh --upload results/0001_openrouter-anthropic-claude-sonnet-4.json

# 執行後趨勢分析
./scripts/run.sh --model openrouter/anthropic/claude-sonnet-4 --trend

# 執行測試
uv run pytest tests/

# Lint
uv run ruff check .
```

---

## 文件地圖

| 文件 | 說明 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系統架構、元件關係、Mermaid 圖表 |
| [DATA_MODEL.md](./DATA_MODEL.md) | 任務資料模型、結果 JSON 格式、評分結構 |
| [API_SURFACE.md](./API_SURFACE.md) | CLI 介面、評分函式 API、任務格式規格 |
| [DEV_GUIDE.md](./DEV_GUIDE.md) | 開發者上手指南、環境建置、debugging |
| [CODEBASE_MAP.md](./CODEBASE_MAP.md) | 程式碼地圖、模組依賴、改 X 看哪裡 |
| [DISCOVERY_LOG.md](./DISCOVERY_LOG.md) | 探索紀錄、文件落差、TODO/FIXME |

---

## 專案術語表

| 術語 | 定義 |
|------|------|
| **OpenClaw** | LLM agent 框架，本 benchmark 的測試對象（external dependency） |
| **Skill** | OpenClaw 的功能模組，PinchBench 本身就是一個 skill（SKILL.md 定義） |
| **Task** | 一個 benchmark 任務，定義在 `tasks/*.md`，包含 prompt、評分標準、grading code |
| **Transcript** | 一次 agent session 的完整對話歷史（JSONL 格式） |
| **Grading Type** | `automated`（Python code）/ `llm_judge`（LLM 評判）/ `hybrid`（兩者混合） |
| **Judge** | 評判 agent 表現的 LLM，可以是 OpenClaw agent session 或直接 API 呼叫 |
| **Workspace** | Agent 執行任務時的隔離目錄（`/tmp/pinchbench/{run_id}/...`） |
| **Run ID** | 本次 benchmark 的識別碼（4 位數字，如 `0001`） |
| **Model Slug** | 模型 ID 的 URL 安全版本（`/` → `-`，`.` → `-`，全小寫） |
| **fws** | Google Workspace mock server（`@juppytt/fws` npm 套件），GWS 任務測試用 |
| **gws** | Google Workspace CLI，讓 OpenClaw 操作 Gmail/Calendar/Drive 等 |
| **Bootstrap files** | OpenClaw agent 的身份/人格定義檔案（`SOUL.md`, `BOOTSTRAP.md` 等） |
| **Fail-fast** | sanity task 得 0 分時中止 benchmark，避免浪費資源 |
| **Manifest** | `tasks/manifest.yaml`，任務有序清單的單一真實來源 |
| **Efficiency metrics** | 分數/token 數、分數/美元，衡量模型的成本效率 |
| **Incremental results** | 每完成一個任務立即寫入 JSON，供外部工具 polling 進度 |
