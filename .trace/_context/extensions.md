# Stage 2.4 — Extension Points

## 1. 新增任務（最主要的擴展點）

這是 PinchBench 設計的**核心擴展機制**。

### 步驟

1. **建立任務檔案**：在 `tasks/` 下建立 `task_my_new_task.md`，遵循 `TASK_TEMPLATE.md` 格式
2. **註冊到 manifest**：在 `tasks/manifest.yaml` 的 `tasks:` 清單末尾加上 `- task_my_new_task`

**CI 驗證**：`scripts/lint_manifest.py` 在 CI 中驗證：
- 每個 `task_*.md` 都出現在 manifest 中
- manifest 中的每個 ID 都有對應的 `.md` 檔案

### 任務可配置的維度

| 欄位 | 說明 |
|------|------|
| `grading_type` | `automated` / `llm_judge` / `hybrid` |
| `timeout_seconds` | 任務執行超時（秒） |
| `workspace_files` | 任務前置 fixture 檔案清單 |
| `grading_weights` | hybrid 模式下 automated vs llm_judge 的權重 |
| `sessions` | multi-turn 對話列表 |
| `prerequisites` | 所需外部工具（如 `npm:@juppytt/fws`） |
| `category` | 任務分類（影響 fws 啟動：gws/github） |

### 新增 fixture 檔案

將素材放在 `assets/`，在任務 frontmatter 中引用：
```yaml
workspace_files:
  - source: assets/my_data.csv
    dest: data.csv
```

---

## 2. 新增 Judge 後端（中等擴展）

**位置**: `lib_agent.py:call_judge_api()` at line 1075

當前 dispatch 邏輯：
```python
def call_judge_api(*, prompt, model, timeout_seconds) -> Dict:
    if model == "claude" or model.startswith("claude:"):
        return _judge_via_claude_cli(...)
    if model.startswith("anthropic/"):
        return _judge_via_anthropic(...)
    if model.startswith("openai/"):
        return _judge_via_openai(...)
    return _judge_via_openrouter(...)  # default
```

要新增 judge 後端，在此函式中添加新的 prefix 分支，並實作對應的 `_judge_via_xxx()` 函式。

---

## 3. 自訂模型 Endpoint（使用者擴展）

`--base-url` flag 啟用自訂 OpenAI-compatible API：

```python
# lib_agent.py:309–338
if base_url:
    providers["custom"] = {
        "baseUrl": base_url,
        "apiKey": key_ref,
        "api": "openai-completions",
        "models": [{
            "id": model_id,
            "contextWindow": 200000,
            "maxTokens": 8192,
        }]
    }
```

這讓本地部署的模型（如 ollama、vLLM）也能被 benchmark。

---

## 4. fws 擴展（GWS 任務類型）

**位置**: `lib_fws.py`

GWS 任務通過設定 `category: gws` 或 `category: github`（或 `prerequisites: [fws]`）自動觸發 fws 生命週期管理：

```python
# lib_fws.py:25
def is_fws_task(frontmatter: dict) -> bool:
    if frontmatter.get("category") in ("gws", "github"):
        return True
    prereqs = frontmatter.get("prerequisites", [])
    return any("fws" in str(p) for p in prereqs)
```

新增需要其他 mock server 的任務類型時，可參考此模式擴展 `lib_fws.py`。

---

## 5. 評分函式（Automated Checks 擴展）

每個任務的評分函式是完全可自訂的 Python code：

```python
def grade(transcript: list, workspace_path: str) -> dict:
    # transcript: JSONL events (type/message/role/content/toolCall 等)
    # workspace_path: 任務執行後的工作區路徑
    # 回傳: {"criterion_key": score_0_to_1, ...}
```

**可用資料**：
- **Transcript**: 完整的 agent 對話歷史（工具呼叫、回應、結果）
- **Workspace 檔案**: agent 在工作區建立的所有檔案
- **注入的 namespace**: `_PINCHBENCH_PRIVATE_IMAGE_KEY_PATH`（圖片分類任務用）

**評分策略**：
- 每個 criterion 獨立評分（0.0–1.0）
- 最終分數 = 所有 criterion 的**算術平均**（非加權）
- 可給部分分數（如 0.5）

---

## 6. CI 工作流擴展

`.github/workflows/` 包含 3 個工作流，可擴展：
- `lint.yml`：新增 lint 規則
- `release.yml`：修改發布流程
- `update-task-count.yml`：任務數自動計數

`.github/benchmark-models.yml`：新增要測試的模型只需加一行。

---

## 7. 環境變數覆蓋（進階調整）

```bash
PINCHBENCH_MAX_MSG_CHARS=8000      # 單次 openclaw message 最大字元數（lib_agent.py:33）
PINCHBENCH_JUDGE_MAX_MSG_CHARS=3000 # judge 訊息最大字元數（lib_agent.py:34）
OPENCLAW_PATH=openclaw              # openclaw CLI 路徑（lib_agent.py:1002）
PINCHBENCH_SERVER_URL=...           # 自訂 API server URL（lib_upload.py:64）
```
