# Stage 1 偵察報告

## 專案概覽

PinchBench 是一套針對 LLM 模型的「真實世界」benchmark 系統，衡量模型作為 OpenClaw agent 大腦的實際表現。由 Kilo.ai（KiloClaw 的製造商）開發，MIT 授權開源。

- **版本**: 2.0.0-rc1（SKILL.md），BENCHMARK_VERSION 檔案記錄 build 版本
- **官網**: https://pinchbench.com（公開排行榜）
- **倉庫**: https://github.com/pinchbench/skill

---

## 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | ≥3.10 | 主要執行環境 |
| Package manager | uv | latest | 依賴管理與腳本執行 |
| Build system | setuptools + setuptools-scm | ≥61 / ≥8 | 打包 |
| Config parser | PyYAML | ≥6.0.1 | 任務 frontmatter 解析 |
| SSH/remote | fabric + paramiko | ≥3.2.2 / ≥3.0.0 | ⚠️ 未驗證實際用途（dependency 存在但程式碼未見直接引用） |
| Linting | ruff | 0.15.6 | CI lint |
| Testing | pytest + pytest-cov | ≥7.4.0 | 單元測試 |
| Format | black | ≥23.7.0 | 程式碼格式化 |
| External CLI | openclaw | runtime dependency | Agent 管理與任務執行 |
| External CLI | fws | optional | Google Workspace mock server |
| External CLI | gws | optional | Google Workspace CLI（GWS 任務用） |
| External API | OpenRouter | - | LLM 模型路由（主要） |
| External API | Anthropic API | - | 直接 judge 模式 |
| External API | OpenAI API | - | 直接 judge 模式 |
| External API | api.pinchbench.com | - | 結果上傳與 token 註冊 |
| CI/CD | GitHub Actions | - | lint / release / task count 更新 |

---

## 目錄結構（3 層）

```
pinchbench-skill/
├── scripts/                    # 核心 Python 程式碼
│   ├── benchmark.py            # 主入口（927 行）
│   ├── lib_agent.py            # OpenClaw 互動層（1242 行）
│   ├── lib_tasks.py            # 任務載入/解析（219 行）
│   ├── lib_grading.py          # 評分引擎（717 行）
│   ├── lib_upload.py           # 排行榜上傳（434 行）
│   ├── lib_fws.py              # GWS mock server 生命週期（105 行）
│   ├── lib_trend.py            # 趨勢分析（177 行）
│   ├── lint_argparse_help.py   # CI 靜態分析（75 行）
│   ├── lint_manifest.py        # manifest 驗證（85 行）
│   └── run.sh                  # uv wrapper（8 行）
├── tasks/                      # 83 個 benchmark 任務定義（Markdown）
│   ├── manifest.yaml           # 任務有序列表（單一真實來源）
│   ├── TASK_TEMPLATE.md        # 任務撰寫模板
│   └── task_*.md               # 各任務定義（81 個）
├── assets/                     # 任務 fixture 檔案
│   ├── csvs/                   # CSV 資料集（8 個）
│   ├── images/                 # 圖片 fixture
│   ├── meetings/               # 會議逐字稿（4 個）
│   ├── logs/                   # Log 檔案（4 個）
│   ├── refactor/               # 重構任務 fixture
│   └── *.py / *.html / *.pdf  # 各類任務素材
├── tests/                      # Python 單元測試
│   ├── test_lib_grading.py     # 評分邏輯測試
│   └── test_lib_trend.py       # 趨勢分析測試
├── .github/
│   ├── benchmark-models.yml    # 預設測試模型清單（35 個）
│   └── workflows/
│       ├── lint.yml            # Ruff + argparse + manifest lint
│       ├── release.yml         # 發布流程
│       └── update-task-count.yml # 自動更新 README 任務數
├── pyproject.toml              # 專案配置
├── SKILL.md                    # OpenClaw skill manifest（YAML + Markdown）
├── README.md                   # 公開文件
├── BENCHMARK_VERSION           # Build 版本號
├── Dockerfile.benchmark        # 容器化 benchmark 環境
├── crab.txt                    # ASCII art（啟動畫面）
└── .pre-commit-config.yaml     # Pre-commit hooks（ruff）
```

---

## 架構模式

**類型**: CLI tool（非 API server、非 library、非前端）
- 屬於 **monolith**，所有邏輯集中在 `scripts/` 下
- 無 web framework、無 ORM、無 message queue
- 依賴外部 `openclaw` CLI（不包含在此 repo）

---

## 既有文件掃描

### 存在的文件
1. **README.md** — 完整的使用說明、Quick Start、Command Reference、Contributing
2. **SKILL.md** — OpenClaw skill manifest 格式（YAML frontmatter + Markdown）
3. **tasks/TASK_TEMPLATE.md** — 任務撰寫完整指南（含 grading function 範例）
4. **assets/csvs/README.md** — CSV 資料集說明
5. **assets/logs/README.md** — Log 檔案說明

### 落差分析（文件 vs. 程式碼）

| 文件說 | 程式碼實際是 | 位置 |
|--------|------------|------|
| SKILL.md 列出 23 個任務 | manifest.yaml 實際有 83 個任務 | `tasks/manifest.yaml` |
| README.md Quick Start 用 `./scripts/run.sh` | SKILL.md Quick Start 用 `uv run benchmark.py`（無 scripts/ 前綴） | SKILL.md:27 |
| README.md 說 53 個任務 | 程式碼實際有 83 個（manifest.yaml 計數） | `tasks/manifest.yaml` |
| `--judge` 預設說 "openrouter/anthropic/claude-opus-4.5" | DEFAULT_JUDGE_MODEL 也是此值 | `lib_grading.py:23` |

---

## 規模評估

- **總檔案數**: 209（遠低於 500）
- **Python 程式碼行數**: ~3,981（scripts/）
- **任務數**: 83
- **結論**: 中小型專案，無需縮限 trace 範圍
