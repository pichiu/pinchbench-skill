# Stage 2.6 — 設定與環境

## 環境變數

### 必要（視使用情境而定）

| 變數 | 用途 | 預設值 |
|------|------|--------|
| `OPENROUTER_API_KEY` | 模型驗證 + OpenRouter judge | 無（未設定時跳過驗證） |
| `ANTHROPIC_API_KEY` | `anthropic/*` judge 直呼 API | 無 |
| `OPENAI_API_KEY` | `openai/*` judge 直呼 API + 自訂 endpoint | 無 |
| `PINCHBENCH_TOKEN` | 結果上傳到排行榜 | 無（未設定時上傳失敗） |

### 可選

| 變數 | 用途 | 預設值 |
|------|------|--------|
| `PINCHBENCH_OFFICIAL_KEY` | 標記官方提交（等同 `--official-key`） | 無 |
| `PINCHBENCH_SERVER_URL` | 自訂排行榜 API URL | `https://api.pinchbench.com` |
| `PINCHBENCH_MAX_MSG_CHARS` | 單次 openclaw message 最大字元數 | `8000` |
| `PINCHBENCH_JUDGE_MAX_MSG_CHARS` | Judge message 最大字元數（超過則分段傳送）| `3000` |
| `OPENCLAW_PATH` | openclaw CLI 路徑 | `openclaw`（PATH 解析） |
| `NO_COLOR` | 停用 ANSI 顏色輸出 | 未設定（有 tty 時啟用） |

### GWS 任務（由 fws 自動設定）

| 變數 | 用途 |
|------|------|
| `GOOGLE_WORKSPACE_CLI_CONFIG_DIR` | gws CLI 配置目錄（指向 mock） |
| `GOOGLE_WORKSPACE_CLI_TOKEN` | mock token |
| `HTTPS_PROXY` | 代理到 fws mock server |
| `SSL_CERT_FILE` | fws 的 CA 憑證 |

---

## 設定檔

### `scripts/.pinchbench/config.json`

**位置**: `lib_upload.py:21`（`CONFIG_PATH = Path(__file__).parent / ".pinchbench" / "config.json"`）

自動建立，通過 `--register` 流程寫入：

```json
{
  "token": "pbk_xxxxxxxxxxxxxxxxxxxxxxxx",
  "claim_url": "https://pinchbench.com/claim?token=..."
}
```

### `~/.openclaw/agents/{agent_id}/agent/models.json`

OpenClaw agent 的模型配置。benchmark.py 在建立 bench agent 時：
- **OpenRouter 模式**：複製 `~/.openclaw/agents/main/agent/models.json`，覆寫 `defaultProvider`/`defaultModel`
- **自訂 endpoint 模式**：建立包含 custom provider 的新 JSON

```json
{
  "defaultProvider": "anthropic",
  "defaultModel": "claude-sonnet-4",
  "models": {
    "mode": "merge",
    "providers": { ... }
  }
}
```

---

## 命令列參數 vs. 環境變數優先順序

```
命令列 --official-key > 環境變數 PINCHBENCH_OFFICIAL_KEY

Token 解析:
  函式參數 token > PINCHBENCH_TOKEN env > .pinchbench/config.json

Server URL:
  函式參數 server_url > PINCHBENCH_SERVER_URL env > "https://api.pinchbench.com"

Judge 設定:
  --judge 設定時 → judge_backend="api"（直接呼叫）
  --judge 未設定時 → judge_backend="openclaw"（透過 agent session）
```

---

## 執行時路徑

| 路徑 | 說明 |
|------|------|
| `/tmp/pinchbench/{run_id}/agent_workspace/` | Agent 工作區（benchmark 執行期間） |
| `/tmp/pinchbench/judge/workspace/` | Judge agent 工作區 |
| `/tmp/pinchbench/judge/private/image_classification_answer_key.json` | 圖片分類答案（執行時複製） |
| `~/.openclaw/agents/{agent_id}/` | OpenClaw agent 儲存目錄 |
| `~/.openclaw/agents/{agent_id}/sessions/` | Session transcript 目錄 |
| `~/.openclaw/agents/{agent_id}/agent/models.json` | 模型配置 |
| `~/.openclaw/workspace/skills/` | 主工作區 skills（benchmark 時複製到 agent workspace） |
| `{output_dir}/{run_id}_{model_slug}.json` | 最終結果 JSON（預設 `results/`） |
| `{output_dir}/{run_id}_transcripts/{task_id}.jsonl` | 各任務的 transcript archive |

---

## Benchmark Version 解析邏輯

`_get_benchmark_version()` 的優先順序（`benchmark.py:316`）：

1. `importlib.metadata.version("pinchbench")`（pip 安裝時）
2. `BENCHMARK_VERSION` 檔案（repo 根目錄）
3. `git describe --tags --long`（git semver tag）
4. `git rev-parse --short HEAD`（commit hash）
5. 空字串（fallback）

---

## 日誌設定

```python
# benchmark.py:41
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.StreamHandler(sys.stdout),
        logging.FileHandler("benchmark.log"),  # 固定路徑，工作目錄下
    ],
)
```

- 同時輸出到 stdout 和 `benchmark.log`
- 無法通過環境變數調整 log level（hardcoded INFO）
- `--verbose` 只影響 logger.info 的詳細程度，不改變 level

---

## 執行 ID 管理

```python
# benchmark.py:290
def _next_run_id(run_root: Path) -> str:
    run_root = Path("/tmp/pinchbench")
    existing = [int(e.name) for e in run_root.iterdir() if e.is_dir() and e.name.isdigit()]
    return f"{max(existing) + 1:04d}"  # 如: "0001", "0002"
```

Run ID 從 `/tmp/pinchbench/` 的現有子目錄數值推算，輸出檔名格式：`{run_id}_{model_slug}.json`。
