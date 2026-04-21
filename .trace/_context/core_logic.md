# Stage 2.3 — 核心領域邏輯

## 系統「心臟」

PinchBench 的核心是**任務評分引擎**（`lib_grading.py`）+ **任務定義格式**（`tasks/*.md`）。這兩者組合定義了「什麼算成功」的邏輯。

---

## 1. 任務定義系統

### 任務格式（Markdown + YAML frontmatter）

每個任務是一個 Markdown 檔案，格式為：

```
---
id: task_calendar
name: Calendar Event Creation
category: calendar
grading_type: automated | llm_judge | hybrid
timeout_seconds: 120
workspace_files:
  - source: assets/sample.csv
    dest: data.csv
grading_weights:          # hybrid only
  automated: 0.5
  llm_judge: 0.5
sessions:                 # optional: multi-turn
  - "Session 1 prompt"
  - "Session 2 prompt"
prerequisites:            # optional: fws/cli
  - npm:@juppytt/fws
---

## Prompt
## Expected Behavior
## Grading Criteria
## Automated Checks    # Python code block
## LLM Judge Rubric   # Markdown rubric
```

### TaskLoader 解析邏輯（lib_tasks.py:138）

```python
def load_task(self, task_file: Path) -> Task:
    # 1. regex 提取 YAML frontmatter
    frontmatter_match = re.match(r"^---\s*\n(.*?)\n---\s*\n(.*)$", content, re.DOTALL)
    # 2. yaml.safe_load() 解析 metadata
    # 3. _parse_sections() 按 ## 標題分割 body
    # 4. _extract_grading_criteria() 從 checklist 格式提取條件
    # 5. 建立 Task 物件
```

關鍵：`automated_checks` 儲存 "Automated Checks" section 的**原始文字**（含 ```python 包裹），在評分時再用 regex 提取並 `exec()`。

---

## 2. 評分引擎（lib_grading.py）

### 三種評分模式

```python
# lib_grading.py:48
def grade_task(...) -> GradeResult:
    if grading_type == "automated":
        return _grade_automated(...)
    if grading_type == "llm_judge":
        return _grade_llm_judge(...)
    if grading_type == "hybrid":
        auto = _grade_automated(...)
        llm  = _grade_llm_judge(...)
        return _combine_grades(task, auto, llm)
```

#### Automated 評分

```python
# lib_grading.py:99
grading_code = _extract_grading_code(task)   # regex 提取 ```python...```
exec(grading_code, namespace)
grade_func = namespace.get("grade")
scores = grade_func(transcript, workspace_path)
total = _average_scores(scores)              # 所有值的算術平均
```

- `grade()` 函式接收 `transcript: list` 和 `workspace_path: str`
- 回傳 `dict[str, float]`（0.0–1.0），每個 key 是一個評分標準
- **最終分數 = 所有 score 的算術平均**（非加權）
- 函式在 isolated namespace 中執行（僅注入 `_PINCHBENCH_PRIVATE_IMAGE_KEY_PATH`）

#### LLM Judge 評分

```python
# lib_grading.py:181
transcript_summary = _summarize_transcript(transcript)
workspace_content = _read_workspace_files(execution_result["workspace"])
rubric = task.llm_judge_rubric or _format_grading_criteria(task)
prompt = _build_judge_prompt(task, transcript_summary, rubric, workspace_content)
```

Judge prompt 的關鍵指令：
- 「你是一個評分函式，**只能輸出 JSON**，不能使用任何工具」
- 「平均分約 0.6–0.7 代表可接受，1.0 需要真正卓越的表現」
- 要求回傳格式：`{"scores": {...}, "total": 0.0, "notes": "..."}`

Judge 呼叫有 2 次重試（`max_judge_attempts = 2`），失敗間隔指數退避（`2^attempt` 秒）。

Judge 後端選擇：
- `judge_backend="openclaw"` → 以 OpenClaw agent session 執行（預設）
- `judge_backend="api"` → 直接呼叫 LLM API（`--judge` flag 啟用）

#### Judge 回應解析（`_parse_judge_response` / `_parse_judge_text`）

健壯的多層次解析策略：
1. 嘗試直接 `json.loads()` 整個回應
2. 嘗試提取 ````json...```` code block
3. 追蹤 brace 深度提取所有 JSON 候選，從**最後一個**開始嘗試
4. 正則表達式 fallback：`(?:total|overall|final)\s*(?:score)?:\s*(0\.\d+)`
5. 全失敗時回傳 `{}`，記 0 分並記錄 notes

#### 回應正規化（`_normalize_judge_response`）

處理各種 LLM 可能回傳的格式變體：
- `{"scores": {...}, "total": 0.9}` → 標準格式
- `{"criteria_scores": {...}}` → Claude 的替代格式
- `{"score": 0.9}` → 簡化格式
- `{"total": 3.85, "scores": {...各為0.x...}}` → 誤加總的情況，自動均值化

```python
# lib_grading.py:699–707
if (total > 1.0 and all(0.0 <= v <= 1.0 for v in values)):
    result["total"] = sum(values) / len(values)  # 修正誤加總
```

#### Hybrid 評分

```python
# lib_grading.py:317
def _combine_grades(task, auto_result, llm_result) -> GradeResult:
    weights = task.grading_weights or {"automated": 0.5, "llm_judge": 0.5}
    combined_score = (auto * auto_weight + llm * llm_weight) / total_weight
```

---

## 3. Workspace 準備邏輯

```python
# lib_agent.py:404
def prepare_task_workspace(skill_dir, run_id, task, agent_id) -> Path:
    workspace = _get_agent_workspace(agent_id)  # 從 openclaw agents list 取得
    # 保存 bootstrap 檔案 → rmtree → 恢復 bootstrap → 複製 fixture → 複製 skills
```

**Bootstrap 檔案**（`SOUL.md`, `BOOTSTRAP.md` 等）是 OpenClaw agent 的身份/人格定義，**不能被清除**，因此在清空工作區前備份、清空後恢復。

---

## 4. Transcript 發現機制（最複雜的邏輯）

**問題**：OpenClaw 忽略我們傳入的 `--session-id`，自行生成 UUID-based session ID，所以我們無法直接知道 transcript 存在哪個路徑。

**解決策略**（`_load_transcript` at lib_agent.py:588，最多重試 15 次）：
1. 讀取 `sessions.json` → 解析真實 session UUID → 嘗試對應路徑
2. 解析 `sessions.json` 的 string values 中包含 `.jsonl` 路徑的項目（mtime 過濾）
3. glob `sessions/` 下所有 `.jsonl`，取最新修改（mtime >= task 開始時間 - 5s）
4. 以傳入的 session_id 直接嘗試（幾乎不會成功）

---

## 5. 效率計算邏輯

```python
# benchmark.py:405
def _compute_efficiency_summary(task_entries, grades_by_task_id):
    # score_per_1k_tokens = total_score / (total_tokens / 1000)
    # score_per_dollar = total_score / total_cost_usd
```

這是 PinchBench 的差異化指標：不只看分數，還看**成本效率**。

---

## 6. 趨勢分析（lib_trend.py）

```python
# lib_trend.py:121
slope, intercept = statistics.linear_regression(xs, ys)
regression_detected = slope < self.regression_threshold  # 預設 -0.5%/run
```

使用 Python stdlib 的 `statistics.linear_regression`（Python 3.10+ 新增）進行 OLS 回歸，偵測 benchmark 分數的時間序列回歸趨勢。
