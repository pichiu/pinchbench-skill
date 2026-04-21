# Stage 2.5 — 外部整合

## 1. OpenClaw CLI（核心依賴）

**類型**: 本地 subprocess 呼叫  
**位置**: `lib_agent.py`

### 使用到的命令

| 命令 | 程式碼位置 | 用途 |
|------|----------|------|
| `openclaw agents list` | lib_agent.py:167, 224 | 列出 agents、取得 workspace 路徑 |
| `openclaw agents add {id} --model ... --workspace ...` | lib_agent.py:276 | 建立新 agent |
| `openclaw agents delete {id} --force` | lib_agent.py:267 | 刪除舊 agent |
| `openclaw agent --agent {id} --session-id {sid} --message {prompt}` | lib_agent.py:796, 839 | 執行 agent 任務 |
| `openclaw agent --local ...` | lib_agent.py:806 | GWS 任務：傳遞本地環境變數 |
| `openclaw --version` | lib_upload.py:316 | 取得版本號（metadata 用） |

### 失敗處理

```python
except FileNotFoundError as exc:
    stderr = f"openclaw command not found: {exc}"
    status = "error"
```

- openclaw 不存在 → status="error"，task 得 0 分，繼續下一個任務（不中止整個 benchmark）
- subprocess timeout → `timed_out=True`，status="timeout"

---

## 2. OpenRouter API（模型驗證）

**類型**: HTTP GET  
**Endpoint**: `https://openrouter.ai/api/v1/models/{model_id}`  
**位置**: `lib_agent.py:49`

```python
def validate_openrouter_model(model_id, timeout_seconds=10.0) -> bool:
    # 1. 嘗試 specific model endpoint（快速路徑）
    #    → 404 時 fallback 到全量 catalog
    # 2. 若找不到，提供 "Did you mean" 建議
```

**跳過驗證的情況**：
- `OPENROUTER_API_KEY` 未設定 → 警告後跳過
- 無 `/` 的 model ID（非 OpenRouter 格式）→ 跳過
- 網路錯誤 → 警告後跳過（不中止）
- `--base-url` 設定時 → 完全跳過

---

## 3. PinchBench API（結果上傳）

**類型**: HTTP POST  
**Endpoint**: `https://api.pinchbench.com/api/results`  
**位置**: `lib_upload.py:38`

```python
headers = {
    "X-PinchBench-Token": resolved_token,
    "X-PinchBench-Version": payload["client_version"],
    "X-PinchBench-Official-Key": official_key,  # 若有
}
```

**Token 解析優先順序**（lib_upload.py:286）：
1. 函式參數 `token`
2. 環境變數 `PINCHBENCH_TOKEN`
3. 設定檔 `scripts/.pinchbench/config.json`

**Token 註冊 endpoint**: `POST https://api.pinchbench.com/api/register`

**失敗處理**：
```python
except UploadError as exc:
    logger.warning("Upload failed: %s", exc)  # 只警告，不中止
```
上傳失敗不影響本地結果的儲存。

---

## 4. Anthropic API（直接 Judge）

**類型**: HTTP POST  
**Endpoint**: `https://api.anthropic.com/v1/messages`  
**位置**: `lib_agent.py:1177`

```python
headers = {
    "x-api-key": api_key,           # ANTHROPIC_API_KEY
    "anthropic-version": "2023-06-01",
}
payload = {
    "model": bare_model,
    "max_tokens": 2048,
    "temperature": 0.0,             # 固定為 0（確定性回應）
    "system": _JUDGE_SYSTEM_MSG,
}
```

---

## 5. OpenAI API（直接 Judge）

**類型**: HTTP POST  
**Endpoint**: `https://api.openai.com/v1/chat/completions`  
**位置**: `lib_agent.py:1165`

使用共用的 `_judge_via_openai_compat()` 實作，與 OpenRouter judge 共享邏輯。

---

## 6. OpenRouter API（Judge 路由）

**類型**: HTTP POST  
**Endpoint**: `https://openrouter.ai/api/v1/chat/completions`  
**位置**: `lib_agent.py:1152`

附加 headers：
```python
extra_headers = {
    "HTTP-Referer": "https://pinchbench.com",
    "X-Title": "PinchBench-Judge",
}
```

---

## 7. Claude CLI（直接 Judge）

**類型**: subprocess  
**命令**: `claude -p [--model model_name]`  
**位置**: `lib_agent.py:1220`

```python
result = subprocess.run(
    cmd,
    input=f"{_JUDGE_SYSTEM_MSG}\n\n{prompt}",
    capture_output=True,
    text=True,
    timeout=timeout_seconds,
)
```

通過 model 前綴 `claude:` 指定版本，如 `claude:claude-opus-4-5`。

---

## 8. fws CLI（GWS Mock Server）

**類型**: subprocess  
**命令**: `fws server start` / `fws server stop`  
**位置**: `lib_fws.py`

```python
result = subprocess.run(["fws", "server", "start"], timeout=30)
# 解析 output 中的 export KEY=VALUE 行
# 設定 HTTPS_PROXY / SSL_CERT_FILE / GOOGLE_WORKSPACE_CLI_CONFIG_DIR / GOOGLE_WORKSPACE_CLI_TOKEN
```

**可用性檢查**: `shutil.which("fws") is not None`

---

## 整合失敗處理總覽

| 整合 | 失敗影響 | 行為 |
|------|---------|------|
| openclaw 不存在 | 單一任務失敗 | status="error"，繼續下一個 |
| openclaw timeout | 單一任務 timeout | status="timeout"，繼續下一個 |
| OpenRouter 驗證失敗 | 中止整個 benchmark | sys.exit(1) |
| Anthropic/OpenAI API 失敗 | Judge 得 0 分 | 記錄 notes，繼續 |
| PinchBench 上傳失敗 | 跳過上傳 | 警告，本地結果不受影響 |
| fws 啟動失敗 | 警告，任務繼續 | fws=None，不使用 GWS mock |

---

## 網路請求一覽

所有 HTTP 請求使用 Python stdlib `urllib.request`（無 requests 依賴）：

```python
from urllib import error, request
req = request.Request(endpoint, data=body, headers=headers, method="POST")
with request.urlopen(req, timeout=timeout_seconds) as resp:
    data = json.loads(resp.read().decode("utf-8"))
```
