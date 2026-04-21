# Stage 2.1 — Entry Points

## 程式啟動路徑

### 路徑 1：Shell wrapper（推薦方式）

```
scripts/run.sh
  └─ exec uv run scripts/benchmark.py "$@"
```

`scripts/run.sh` 僅 8 行，設定工作目錄為 repo 根目錄後直接委派給 `uv run`。

### 路徑 2：直接執行（SKILL.md 文件說明）

```
uv run benchmark.py --model anthropic/claude-sonnet-4
```

### 路徑 3：安裝後執行（pyproject.toml scripts entry）

```
benchmark --model anthropic/claude-sonnet-4
  └─ [project.scripts] benchmark = "benchmark:main"
```

---

## 主要入口：`main()` 函式

**位置**: `scripts/benchmark.py:589`

### 啟動序列（按順序）

```
main() at benchmark.py:589
│
├─ 1. 確定目錄
│     script_dir = Path(__file__).parent      # scripts/
│     skill_root = script_dir.parent          # repo root
│     tasks_dir = skill_root / "tasks"        # tasks/
│
├─ 2. 顯示 ASCII art
│     _load_ascii_art(skill_root, "crab.txt") → crab.txt
│     _colorize_gradient() → ANSI 漸層色（tty 下）
│     time.sleep(5)                           # 故意延遲 5 秒
│
├─ 3. 解析命令列參數
│     _parse_args() at benchmark.py:174
│     → argparse.Namespace
│
├─ 4. 特殊模式（register / upload）
│     --register → lib_upload.register_token() 後 return
│     --upload   → lib_upload.upload_results() 後 return
│
├─ 5. 載入任務
│     BenchmarkRunner(tasks_dir).load_tasks()
│       └─ TaskLoader.load_all_tasks()
│            └─ _load_from_manifest(manifest.yaml)  # 若存在
│                 或 _load_from_glob()               # fallback
│
├─ 6. 驗證模型
│     validate_openrouter_model(args.model)
│     → HTTP GET https://openrouter.ai/api/v1/models/{model_id}
│
├─ 7. 建立/確保 OpenClaw agent
│     ensure_agent_exists(agent_id, args.model, agent_workspace)
│       ├─ subprocess: openclaw agents list
│       ├─ subprocess: openclaw agents delete {agent_id} --force (若需重建)
│       ├─ subprocess: openclaw agents add {agent_id} --model ... --workspace ...
│       └─ 設定 models.json（provider/model 配置）
│
├─ 8. 清理舊 session
│     cleanup_agent_sessions(agent_id)
│     → 刪除 ~/.openclaw/agents/{agent_id}/sessions/ 下的 *.jsonl
│
└─ 9. 主迴圈（每個任務）
      for task in tasks_to_run:
        for run_index in range(runs_per_task):
          execute_openclaw_task(...)
          grade_task(...)
          _write_incremental_results()
      → _build_and_write_results()
      → _log_category_summary()
      → upload_results()（若未設 --no-upload）
```

---

## 關鍵初始化細節

### TaskLoader 初始化

```python
# lib_tasks.py:83
def load_all_tasks(self) -> List[Task]:
    manifest_path = self.tasks_dir / "manifest.yaml"
    if manifest_path.exists():
        return self._load_from_manifest(manifest_path)
    return self._load_from_glob()
```

**manifest.yaml** 是任務順序的唯一真實來源。`task_sanity` 排在第一位，若其得分為 0 且有 transcript，會觸發 fail-fast（`sys.exit(3)`），節省後續資源。

### Agent 配置

```python
# lib_agent.py:304–364
# 決策樹：
# 若 --base-url → 建立 custom OpenAI-compatible provider
# 否則 → 複製 main agent 的 models.json，覆寫 defaultProvider/defaultModel
```

Agent 儲存路徑: `~/.openclaw/agents/{agent_id}/`（`bench-{model_slug}` 格式）

---

## 命令列參數總覽

| 參數 | 預設值 | 說明 |
|------|--------|------|
| `--model MODEL` | 必填（除非 --register/--upload） | 模型 ID（需含 provider prefix） |
| `--suite SUITE` | `all` | `all`, `automated-only`, 或逗號分隔 task ID |
| `--output-dir DIR` | `results/` | 結果輸出目錄 |
| `--runs N` | `1` | 每個任務執行次數（用於平均） |
| `--timeout-multiplier N` | `1.0` | 縮放所有任務 timeout |
| `--judge MODEL` | `None`（使用 OpenClaw session） | judge 模型 |
| `--base-url URL` | `None` | 自訂 OpenAI-compatible endpoint |
| `--api-key KEY` | `None`（用 OPENAI_API_KEY） | 自訂 endpoint API key |
| `--no-upload` | `False` | 跳過上傳排行榜 |
| `--register` | `False` | 申請 API token |
| `--upload FILE` | `None` | 上傳既有結果 JSON |
| `--official-key KEY` | `None` | 標記官方提交 |
| `--no-fail-fast` | `False` | 即使 sanity=0 也繼續 |
| `--verbose / -v` | `False` | 詳細 log |
| `--trend` | `False` | 執行後進行趨勢分析 |
| `--trend-window N` | `10` | 趨勢分析的 run 窗口數 |
| `--trend-threshold N` | `-0.5` | 回歸偵測門檻（%/run） |
