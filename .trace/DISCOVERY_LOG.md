# DISCOVERY_LOG.md — PinchBench 探索紀錄與待解問題

> 產出日期：2026-04-21  
> 版本：2.0.0-rc1  
> 探索範圍：`scripts/`、`tasks/`、`assets/`、`.github/`、設定檔  

---

## 1. Web Search 發現摘要

### 官方資源

| 資源 | URL | 關鍵 Takeaway |
|------|-----|--------------|
| GitHub 主倉庫 | https://github.com/pinchbench/skill | 本 repo，MIT 授權 |
| 排行榜網站 | https://pinchbench.com/ | 公開排行榜，按成功率/速度/成本排序 |
| About / FAQ | https://pinchbench.com/about | 評分方式說明 |
| GitHub 組織 | https://github.com/pinchbench | 含 leaderboard repo（UI 分離） |
| Kilo Blog v2 公告 | https://blog.kilo.ai/p/pinchbench-v2-call-for-contributors | v2 貢獻徵集，截止 2026-04-15 |

### 競品與相關資源

| 資源 | URL | 關鍵 Takeaway |
|------|-----|--------------|
| LiveClawBench | https://arxiv.org/html/2604.13072 | Triple-Axis Complexity Framework，學術方法 |
| arthursoares/openclaw-llm-bench | GitHub（⚠️ 未驗證） | 52 prompts、11 traps、LLM-as-judge |
| pricepertoken.com | https://pricepertoken.com/leaderboards/openclaw | 社群票選排名 |
| OpenClaw.report | https://openclaw.report | 收錄 PinchBench 介紹的生態系統頁面 |
| fws npm package | https://www.npmjs.com/package/@juppytt/fws | GWS mock server，⚠️ 無官方文件 |

### 排行榜現狀（截至 2026-04-21）

- 最高成功率約 **中 80% 範圍**，顯示 agentic 任務仍具挑戰
- 社群票選最佳模型：**Kimi K2.5** > GLM 4.7 > Claude Opus 4.6
- 計分三維度：成功率（Success Rate）、速度（Speed）、成本效率（Cost）

---

## 2. 文件與程式碼落差清單

### SKILL.md 與實際任務數

| 項目 | SKILL.md 記載 | 程式碼實際 | 落差 |
|------|--------------|-----------|------|
| 任務數量 | 23 個任務（列表） | manifest.yaml 有 76 個 task_id 項目，實體 .md 檔案 76 個 | **少列 53 個任務** |
| Quick Start 路徑 | `uv run benchmark.py`（無 `scripts/` 前綴） | 實際應為 `uv run scripts/benchmark.py` 或透過 `./scripts/run.sh` | 路徑不一致 |

### README.md 與實際情況

| 項目 | README 記載 | 實際情況 | 落差 |
|------|------------|---------|------|
| 任務數量 badge | `tasks-53-orange` | 實際有 76 個 `.md` 檔案（manifest 有 76 條） | **少計 23 個** |
| 任務數量文字 | "PinchBench includes 53 tasks" | 同上 | 需更新 |
| 自動更新機制 | `.github/workflows/update-task-count.yml` 存在 | ⚠️ CI workflow 存在但數字未同步到最新 | CI 可能未被觸發 |

> **注意**：`web_findings.md` 記載 v2 目標為 100 個任務、目前 83 個；但本地 manifest.yaml 計數為 76（⚠️ 版本差異或計數方式差異，待確認）

### 其他發現的不一致

- `SKILL.md:27` 的 Quick Start 指令 `uv run benchmark.py` 與 `README.md` 的 `./scripts/run.sh` 風格不同，兩者均有效但容易混淆
- `lib_grading.py:23` 的 `DEFAULT_JUDGE_MODEL = "openrouter/anthropic/claude-opus-4.5"` 與 README 描述相符，無落差
- `BENCHMARK_VERSION` 檔案存在於 repo 根目錄，但 `_get_benchmark_version()` 優先使用 pip metadata，次才讀此檔案——若非 pip 安裝則版本可能不一致

---

## 3. 程式碼中發現的 TODO/FIXME/HACK

**搜尋結果：`scripts/` 目錄下無任何 `TODO`、`FIXME`、`HACK`、`XXX` 標記。**

這意味著：
- 開發者未在程式碼中留下顯式技術債標記
- 潛在問題以「設計決策」形式存在，需從行為推斷（見第 4 節）

---

## 4. 已知技術債

### 4.1 高風險：`exec()` 執行 Grading Code

```python
# scripts/lib_grading.py:117
exec(grading_code, namespace)
```

- **問題**：每個任務的 `## Automated Checks` section 中的 Python code 在評分時以 `exec()` 動態執行
- **風險**：若任務定義檔案（`tasks/*.md`）被惡意修改，可執行任意 Python code
- **現況緩解**：`namespace` 僅注入 `_PINCHBENCH_PRIVATE_IMAGE_KEY_PATH`，但未沙箱化（無 `RestrictedPython` 或等效機制）
- **影響範圍**：所有 `grading_type: automated` 或 `hybrid` 的任務

### 4.2 中風險：Hardcoded Magic Numbers

| 位置 | 數值 | 說明 |
|------|------|------|
| `lib_agent.py:331-332` | `contextWindow: 200000`, `maxTokens: 8192` | 自訂 endpoint 模式下的固定值，不適用於小型模型 |
| `lib_grading.py:225` | `max_judge_attempts = 2` | Judge 重試次數 hardcoded，無法通過環境變數調整 |
| `lib_grading.py:438` | `content[:3000]` | workspace 檔案讀取截斷，大型檔案靜默丟失內容 |
| `lib_agent.py:23` | `DEFAULT_JUDGE_MODEL` | 固定在程式碼中，升級模型需改程式碼 |

### 4.3 中風險：Hardcoded 路徑

| 位置 | 路徑 | 說明 |
|------|------|------|
| `benchmark.py:44` | `benchmark.log` | 固定寫在執行目錄（非 `/tmp`），可能污染工作目錄 |
| `benchmark.py:655` | `/tmp/pinchbench` | Run root 硬編碼為 `/tmp`，Windows 或特殊環境會失敗 |

### 4.4 低風險：設計限制

- **Log level 不可調整**：`logging.basicConfig(level=logging.INFO)` 固定為 INFO，`--verbose` 僅影響 logger 訊息內容而非 level，debug 困難
- **Transcript 發現機制脆弱**：OpenClaw 不遵守傳入的 `--session-id`，`_load_transcript()` 需嘗試 4 種策略、最多重試 15 次（`lib_agent.py:588`）
- **LLM 回應正規化複雜度**：`_normalize_judge_response()` 需處理 5+ 種 LLM 格式變體，顯示 judge prompt 尚未穩定收斂
- **Run ID 依賴 `/tmp` 目錄狀態**：若 `/tmp/pinchbench/` 被清除，run ID 從 `0001` 重新計算，可能覆蓋同名結果檔案

---

## 5. 未解答的疑問

### 5.1 fabric / paramiko 的實際用途

```toml
# pyproject.toml:13-14
"fabric>=3.2.2",
"paramiko>=3.0.0",
```

- `scripts/` 下無任何 `import fabric` 或 `import paramiko`
- 可能用途推測：⚠️ 未驗證
  - 計畫中的「遠端 agent 執行」功能（SSH 到遠端機器跑 benchmark）
  - 歷史遺留依賴（舊版本曾用，現已移除但未清理 `pyproject.toml`）
  - 未來的 `Dockerfile.benchmark` 遠端部署場景
- **建議**：詢問維護者是否可移除，或補充文件說明用途

### 5.2 Dockerfile.benchmark 使用場景

```dockerfile
# Dockerfile.benchmark (35 行)
FROM node:22-bookworm
# 安裝 Python, uv, OpenClaw
ENTRYPOINT ["/bin/bash", "-c"]
CMD ["openclaw --version && echo 'Ready for benchmarks'"]
```

- 無對應的 `docker-compose.yml` 或 CI workflow 引用此 Dockerfile
- 推測用途：⚠️ 未驗證
  - 提供給社群貢獻者的標準化執行環境
  - 排行榜官方評測用的隔離環境
  - 計畫整合進 `release.yml` 但尚未完成
- 已知問題：未 `COPY` 任何 benchmark 程式碼，需使用者自行掛載 volume

### 5.3 fws (`@juppytt/fws`) 的深層行為

- fws 的 GitHub repo 或官方文件未找到
- PinchBench 假設 `fws server start` 會輸出 `export KEY=VALUE` 格式——若 fws 更新 API 將靜默失效
- `lib_fws.py` 依賴 `shutil.which("fws")` 偵測可用性，但未驗證版本相容性

### 5.4 排行榜計分公式

- `api.pinchbench.com` 的伺服器端計分邏輯不在此 repo 中
- 本地計算的分數（算術平均）vs. 排行榜顯示的分數是否相同？⚠️ 未驗證
- 速度（Speed）和成本（Cost）的排名公式未文件化

### 5.5 OpenRouter 驗證失敗行為不對稱

- OpenRouter 驗證失敗 → `sys.exit(1)`（中止整個 benchmark）
- openclaw 不存在 → 單一任務得 0 分（繼續）
- 此非對稱設計是刻意的嗎？（見 `integrations.md`）

---

## 6. 需要更深入調查的區域

### 6.1 任務計數差異

- `recon.md` 說 83 個任務
- 本地 `manifest.yaml` 有 76 個 task_id
- 實體 `.md` 檔案有 76 個
- `web_findings.md` 提到「目前 83 個」（v2 截止前）
- **行動**：確認 manifest.yaml 計數方式 vs. web 資料的時間點差異

### 6.2 GWS 任務的端對端行為

- `lib_fws.py` 只有 105 行，fws 生命週期管理看起來簡單
- 但 fws 本身行為（mock 哪些 GWS API？mock 精準度？）完全不透明
- 影響：`gws_*` 類別的 3 個任務評分可信度

### 6.3 `_PINCHBENCH_PRIVATE_IMAGE_KEY_PATH` 的安全模型

- 圖片分類答案在執行時複製到 `/tmp/pinchbench/judge/private/`
- 評分時透過 namespace 注入路徑
- 若 agent 在 transcript 中洩漏此路徑，是否影響評分公平性？

### 6.4 Transcript JSONL 格式規範

- `lib_agent.py:588` 的 `_load_transcript()` 讀取 OpenClaw 的 `.jsonl` 檔案
- 但 transcript 的 schema（欄位定義）未在任何文件中說明
- 任務撰寫者需猜測 `type`/`message`/`role`/`toolCall` 等欄位名稱

---

## 7. 與維護者確認的問題清單

| # | 問題 | 優先級 | 相關位置 |
|---|------|--------|---------|
| 1 | `fabric` + `paramiko` 依賴的實際用途是什麼？是否可移除？ | 高 | `pyproject.toml:13-14` |
| 2 | `Dockerfile.benchmark` 的預期使用流程是什麼？是否有配套的 CI job？ | 高 | `Dockerfile.benchmark` |
| 3 | `exec()` 執行 grading code 是否有沙箱計畫？還是設計上信任貢獻者？ | 高 | `lib_grading.py:117` |
| 4 | README/SKILL.md 的任務數字（53/23）為何未同步到最新？`update-task-count.yml` 是否正常運作？ | 中 | `README.md:7`, `SKILL.md` |
| 5 | 本地計算的分數與 `api.pinchbench.com` 排行榜分數是否完全相同？ | 中 | `lib_upload.py` |
| 6 | OpenClaw 忽略 `--session-id` 是已知問題還是版本 bug？有無計畫修復？ | 中 | `lib_agent.py:588` |
| 7 | `contextWindow: 200000` / `maxTokens: 8192` 在 `--base-url` 模式下是否可配置？ | 低 | `lib_agent.py:331-332` |
| 8 | Transcript JSONL 的 schema 文件在哪裡？如何讓任務貢獻者正確使用 transcript？ | 低 | `tasks/TASK_TEMPLATE.md` |
| 9 | `max_judge_attempts = 2` 是否足夠？有無統計數據顯示 judge 失敗率？ | 低 | `lib_grading.py:225` |

---

## 8. 技術債分類圖

```mermaid
quadrantChart
    title 技術債優先級矩陣（影響力 vs. 修復難度）
    x-axis 修復容易 --> 修復困難
    y-axis 影響低 --> 影響高
    quadrant-1 立即處理
    quadrant-2 規劃修復
    quadrant-3 低優先
    quadrant-4 長期評估
    exec() 無沙箱: [0.75, 0.90]
    文件任務數落差: [0.15, 0.65]
    OpenClaw session-id 問題: [0.80, 0.75]
    fabric/paramiko 未使用依賴: [0.10, 0.45]
    benchmark.log 路徑固定: [0.10, 0.35]
    contextWindow 硬編碼: [0.25, 0.40]
    Dockerfile 無配套說明: [0.20, 0.50]
    Log level 不可調: [0.15, 0.25]
    Transcript schema 無文件: [0.35, 0.55]
    Judge 重試次數硬編碼: [0.20, 0.30]
```

---

*本文件根據靜態程式碼分析與 web 搜尋結果產出。標注 ⚠️ 未驗證 的項目需實際執行或詢問維護者確認。*
