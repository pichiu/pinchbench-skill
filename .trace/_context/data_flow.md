# Stage 2.2 — Request / Data Flow

## 代表性 Use Case：執行 `task_calendar`

```bash
./scripts/run.sh --model openrouter/anthropic/claude-sonnet-4 --suite task_calendar
```

---

## 完整 Flow 追蹤

```
scripts/run.sh
  └─ uv run scripts/benchmark.py --model ... --suite task_calendar
        └─ main() [benchmark.py:589]
```

### Layer 1：任務選擇

```python
# benchmark.py:681–688
task_ids = _select_task_ids(runner.tasks, args.suite)
# "task_calendar" → task_ids = ["task_calendar"]
tasks_to_run = [task for task in runner.tasks if task.task_id in task_ids]
```

### Layer 2：任務執行（execute_openclaw_task）

```python
# lib_agent.py:730
result = execute_openclaw_task(
    task=task,
    agent_id="bench-openrouter-anthropic-claude-sonnet-4",
    model_id="openrouter/anthropic/claude-sonnet-4",
    run_id="0001-1",
    timeout_multiplier=1.0,
    skill_dir=skill_root,
    output_dir=Path("results/0001_transcripts"),
)
```

#### 2a：工作區準備

```python
# lib_agent.py:404
workspace = prepare_task_workspace(skill_dir, run_id, task, agent_id)
```

步驟：
1. 取得 agent workspace 路徑（`openclaw agents list` 解析）
2. 保存 bootstrap 檔案（`SOUL.md`, `BOOTSTRAP.md`, `USER.md`, `IDENTITY.md`, `HEARTBEAT.md`, `TOOLS.md`）
3. `shutil.rmtree(workspace)` 清除舊內容
4. 恢復 bootstrap 檔案
5. 複製 `task.workspace_files` 中的 fixture（`task_calendar` 的 workspace_files 為空）
6. 複製 `~/.openclaw/workspace/skills/` 到工作區

#### 2b：OpenClaw 子程序呼叫

```python
# lib_agent.py:840
result = subprocess.run(
    ["openclaw", "agent",
     "--agent", "bench-openrouter-anthropic-claude-sonnet-4",
     "--session-id", "task_calendar_1234567890",
     "--message", task.prompt],  # 完整 prompt 文字
    capture_output=True,
    text=True,
    cwd=str(workspace),
    timeout=120.0,  # task_calendar.timeout_seconds = 120
)
```

`openclaw agent` 命令：
- 以指定 model 執行 agent session
- Agent 收到 prompt，使用工具（建立 ICS 檔案等）
- Session transcript 寫入 `~/.openclaw/agents/{agent_id}/sessions/{uuid}.jsonl`

#### 2c：Transcript 載入

```python
# lib_agent.py:863
transcript, transcript_path = _load_transcript(agent_id, session_id, start_time)
```

重試策略（最多 15 次，每次 sleep 1s）：
1. 讀取 `sessions.json` 取得實際 session UUID
2. 嘗試 `sessions/{uuid}.jsonl` 等路徑
3. glob 最新修改的 `.jsonl`
4. 以傳入的 session_id 直接嘗試

#### 2d：Token 使用量擷取

```python
# lib_agent.py:699
usage = _extract_usage_from_transcript(transcript)
# 累加所有 assistant message 的 usage.input/output/totalTokens/cost
```

### Layer 3：評分（grade_task）

```python
# lib_grading.py:48
grade = grade_task(task=task, execution_result=result, skill_dir=skill_dir)
```

`task_calendar` 的 `grading_type = "automated"` → `_grade_automated()`

```python
# lib_grading.py:99
grading_code = _extract_grading_code(task)
# 從 task.automated_checks（"Automated Checks" section 的內容）提取 ```python...``` 區塊

namespace = _build_automated_namespace(skill_dir)
exec(grading_code, namespace)
grade_func = namespace.get("grade")
scores = grade_func(
    execution_result["transcript"],   # List[Dict]
    execution_result["workspace"],    # "/tmp/pinchbench/0001-1/.../workspace"
)
# 回傳例如: {"file_created": 1.0, "date_correct": 0.5, ...}
total = _average_scores(scores)  # 所有 score 的平均值
```

### Layer 4：結果聚合與寫入

```python
# benchmark.py:817–824
grades_by_task_id[task.task_id] = {
    "runs": [grade.to_dict() for grade in task_grades],
    "mean": statistics.mean(task_scores),
    "std": statistics.stdev(task_scores) if len(task_scores) > 1 else 0.0,
    "min": min(task_scores),
    "max": max(task_scores),
}
_write_incremental_results()  # 每次任務完成後寫入中間結果
```

### Layer 5：最終輸出

```python
# benchmark.py:854
task_entries, efficiency = _build_and_write_results()
# 寫入: results/0001_openrouter-anthropic-claude-sonnet-4.json
```

JSON 結構：
```json
{
  "model": "openrouter/anthropic/claude-sonnet-4",
  "benchmark_version": "2.0.1",
  "run_id": "0001",
  "timestamp": 1745123456.789,
  "suite": "task_calendar",
  "runs_per_task": 1,
  "tasks": [
    {
      "task_id": "task_calendar",
      "status": "success",
      "timed_out": false,
      "execution_time": 45.2,
      "transcript_length": 12,
      "usage": {"input_tokens": 1200, "output_tokens": 340, ...},
      "workspace": "/tmp/pinchbench/...",
      "grading": {"runs": [...], "mean": 0.833, "std": 0.0, ...},
      "frontmatter": {...}
    }
  ],
  "efficiency": {
    "total_tokens": 1540,
    "score_per_1k_tokens": 0.541,
    "score_per_dollar": 1250.0,
    ...
  }
}
```

### Layer 6：上傳（可選）

```python
# lib_upload.py:38
result = upload_results(output_path, official_key=args.official_key)
# POST https://api.pinchbench.com/api/results
# Header: X-PinchBench-Token, X-PinchBench-Official-Key
```

---

## Transcript 格式

JSONL 格式，每行一個 event：

```json
{"type": "message", "message": {"role": "user", "content": [...]}}
{"type": "message", "message": {"role": "assistant", "content": [{"type": "text", "text": "..."}, {"type": "toolCall", "name": "write_file", "arguments": {...}}], "usage": {"input": 1200, "output": 340, ...}}}
{"type": "message", "message": {"role": "toolResult", "content": [...]}}
```

---

## Multi-Session 任務 Flow

`task.frontmatter.sessions` 非空時（`sessions: [...]` in frontmatter）：

```python
# lib_agent.py:773–825
for i, session_entry in enumerate(sessions, 1):
    # 每個 session 發送一條訊息，共用同一 session_id
    result = subprocess.run(["openclaw", "agent", ..., "--message", session_prompt])
    elapsed = time.time() - start_time
    if elapsed >= timeout_seconds:
        timed_out = True; break
```

用於需要多輪對話的複雜任務（如 `task_memory` 的記憶測試）。

---

## GWS 任務額外 Flow

當 `is_fws_task(task.frontmatter)` 為 True（category=gws/github）：

```python
# lib_fws.py
fws_env = start_fws()           # subprocess: fws server start
# → 設定 HTTPS_PROXY / SSL_CERT_FILE 等環境變數
# openclaw 呼叫加 --local flag
stop_fws(fws_env)               # subprocess: fws server stop
```
