# Stage 1 線上搜尋結果

## 搜尋查詢清單

1. "PinchBench OpenClaw agent benchmark leaderboard pinchbench.com"
2. "OpenClaw agent framework LLM benchmark 2025"
3. "PinchBench v2 kilo.ai contributors blog 2025 2026"
4. "openclaw agent fws Google Workspace mock server tool"

---

## 關鍵發現

### PinchBench 官方資源

| 資源 | URL | 說明 |
|------|-----|------|
| GitHub 主倉庫 | https://github.com/pinchbench/skill | 本 repo |
| 排行榜網站 | https://pinchbench.com/ | 公開排行榜，按成功率排序 |
| About 頁面 | https://pinchbench.com/about | FAQ |
| GitHub 組織 | https://github.com/pinchbench | 含 leaderboard repo |
| Kilo Blog 宣告 | https://blog.kilo.ai/p/pinchbench-v2-call-for-contributors | v2 貢獻徵集 |

### 關鍵 Takeaways

#### 1. 背景與定位
- PinchBench 由 **Kilo.ai**（KiloClaw 的製造商）開發，目的是幫助用戶從 500+ 模型中選擇最適合 OpenClaw 的 LLM
- OpenClaw 本身是 2025 年 11 月由奧地利開發者 Peter Steinberger 打造的 AI agent 平台（前身 Clawdbot），核心概念是讓 AI 能接管數位工作流程

#### 2. 排行榜現狀（截至 2026 年 4 月）
- 最高分約在 **中 80% 範圍**，顯示 agentic 任務完成仍具挑戰性
- 社群票選最佳模型：Kimi K2.5 > GLM 4.7 > Claude Opus 4.6
- 截至 v1 發布：32 個模型、192 次執行
- 計分維度：成功率（Success Rate）、速度（Speed）、成本（Cost）

#### 3. PinchBench v2 計畫
- 貢獻窗口截止：2026 年 4 月 15 日
- v2 目標：**100 個任務**（目前 83 個）
- 改進方向：更長任務 horizon、更好的驗證、更豐富的領域覆蓋
- 貢獻者分兩類：Skills Contributors（新任務）和 Leaderboard Contributors（UI/UX）

#### 4. Google Workspace (GWS) 整合
- `gws` 是 Google 官方發布的 CLI，讓 OpenClaw 控制 Gmail、Docs、Drive、Calendar、Sheets
- `fws`（@juppytt/fws）是 Google Workspace 的 **mock server**，用於測試時模擬 GWS 環境
  - 以 npm 安裝：`npm install -g @juppytt/fws`
  - 啟動命令：`fws server start`，輸出環境變數（proxy、SSL cert、token）
  - PinchBench 在 `lib_fws.py` 中管理其生命週期
- GWS 任務的 category 標記為 `gws`，`github` category 也使用 fws

#### 5. 競品與相關 benchmarks
- **LiveClawBench** (arxiv.org/html/2604.13072) — 使用 Triple-Axis Complexity Framework
- **arthursoares/openclaw-llm-bench** — 52 prompts、11 traps、LLM-as-judge、tier-based leaderboard
- **pricepertoken.com/leaderboards/openclaw** — 社群票選排名

### 安全研究文獻（觀察）
- 多篇 arXiv 論文研究 OpenClaw 的安全性（ClawSafety、Taming OpenClaw），顯示此 agent 框架在學術界受到關注
- PinchBench 的 `task_cve_security_triage` 任務涵蓋了安全相關評估

---

## 既有討論

- Charly Wargnier 在 X 上的分享（@DataChaz）指出排行榜的互動圖表功能
- Apiyi.com 整理了「OpenClaw + PinchBench 的 5 個關鍵評估維度」指南
- OpenClaw.report 生態系統頁面收錄了 PinchBench 的介紹

---

## 未找到的資訊

- fws (`@juppytt/fws`) 的官方文件或 GitHub repo（搜尋結果未直接指向）
- Dockerfile.benchmark 的使用情境說明
- 排行榜的詳細計分公式
