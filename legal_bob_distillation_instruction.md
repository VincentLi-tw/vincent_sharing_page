# Legal Bob

IBM APAC Legal 團隊內部工具，整合 **Bob Shell**（IBM 內部 AI CLI）來協助：

1. 將歷史法律爭議案件（判決書、合約）蒸餾成結構化知識庫（三層蒸餾架構）
2. 對 MSA / SOW 合約文件執行 APAC MSA Heatmap 風險評估，輸出可分享的 HTML 報告

專案目前只有 **後端 API**（`src/server.js`，Express）。`package.json` 中殘留的 `dev` / `server` / `server:tui` 相關 vite/前端腳本是舊版遺留，倉庫裡沒有對應的前端原始碼（見下方「已知問題」）。

---

## 1. 系統需求

| 項目 | 版本 / 說明 |
|---|---|
| Node.js | 22.x（`Containerfile` 使用 `node:22-bookworm-slim`） |
| Python 3 | 3.9+，需能安裝 `requirements.txt` 內套件 |
| **Bob Shell** | IBM 內部 CLI，需從 `https://bob.ibm.com/download/bobshell.sh` 安裝，並持有有效的 API Key。這是外部人員無法取得的內部工具，**沒有 Bob Shell 這個後端幾乎無法運作**（`src/config/bob.js` 會在啟動時尋找 `bob` binary） |
| Tesseract OCR | 含 `chi-tra` / `chi-sim` 語言包（掃描版合約 OCR 用） |
| LibreOffice | `.docx` 轉換用（僅容器化部署需要，本機開發若不處理 docx 可省略） |
| Podman 或 Docker | 容器化部署用（`compose.yaml` 預設用 `podman compose`；`start.sh` 用的是 `docker-compose` 指令，兩者擇一並自行調整） |

---

## 2. 環境變數與外部設定

### 2.1 `.env`（專案根目錄，**不要提交到 git**，已在 `.gitignore` 中）

```bash
BOBSHELL_API_KEY=你的_Bob_Shell_API_Key
```

### 2.2 `~/.bob/settings.json`（使用者主目錄，非專案內）

Bob Shell 的個人設定檔，包含 API Key 認證與 IBM 實例資訊，啟用 `.bob/custom_modes.yaml` 中定義的 `general-style` / `legal-memo-style` / `heatmap-style` 三個自訂模式。容器化部署時會掛載進容器（見 `compose.yaml`）。

### 2.3 `BOX_BASE_DIR`

案件知識庫（wiki）實際存放在 IBM Box 雲端目錄，本機開發時透過 Box 客戶端同步到本地路徑，再由環境變數指定：

```bash
export BOX_BASE_DIR="/path/to/your/Box-Box/IBM_BOB-Special_Project_Task_Force_for_Legal_Team"
```

⚠️ `src/config/constants.js` 目前把這個路徑的 **fallback 預設值硬編碼成專案原作者本機的絕對路徑**（`/Users/vincent/Library/CloudStorage/...`）。分享專案前務必移除這個 fallback，強制要求使用者自行設定 `BOX_BASE_DIR`，否則其他人在沒有設定環境變數時，程式會嘗試讀寫到一個在你電腦上才存在的路徑。

---

## 3. 安裝與啟動

### 3.1 本機開發

```bash
# 安裝 Node 依賴
npm install

# 安裝 Python 依賴（OCR / 文件轉換用）
pip3 install -r requirements.txt

# 設定環境變數
cp .env.example .env   # 若尚未建立 .env.example，先參考上方 2.1 手動建立
export BOX_BASE_DIR="/path/to/box/wiki"

# 啟動後端 API（唯一可運作的啟動腳本）
npm run server:refactored
# 等同於 node src/server.js，預設監聽 http://localhost:3001
```

驗證：

```bash
curl http://localhost:3001/api/health
```

### 3.2 容器化部署（Podman）

```bash
# 確認 .env 存在，且 Box 目錄可存取
ls -la .env

# 建置映像檔（會在容器內安裝 Bob Shell、LibreOffice、Tesseract）
podman compose build

# 啟動服務
podman compose up -d

# 查看日誌 / 驗證
podman compose logs -f legal-bob-backend
curl http://localhost:3001/api/health
```

完整容器化細節與疑難排解見 `DEPLOYMENT_GUIDE.md`（**注意**：此文件內也寫死了原作者本機路徑，分享前需同步修改，見第 5 節）。

---

## 4. 三層蒸餾架構：如何手動跑完整流程

知識庫的核心設計是「三層蒸餾」，把原始法律文件逐步提煉成高密度、可檢索的知識庫，最終產出一份 LLM 自維護的風險檢查清單。完整設計理念見 `documentations/LLM_WIKI_FLOW.md` 與 `CROSS_CASE_DISTILLATION_ARCHITECTURE.md`。

```
第零層：文件準備       PDF → OCR/轉換 → Markdown
第一層蒸餾：單案例分析  Markdown → wiki/cases/CASE-XXXX/ 的 6 個標準文件
第二層蒸餾：跨案例聚合  多個案例 → wiki/meta/cross-case/（Case Cards、爭議類型、可執行建議）
第三層蒸餾：Meta-Prompt 所有 Case Cards → wiki/meta/system/risk-checklist.md
```

這三層的「Command」**不是 Claude Code 的 slash command**，而是 **Bob Shell** 這套 IBM 內部 CLI 底下的自訂命令，定義在 `.bob/commands/*.md`（Bob Shell 讀取這個資料夾作為 prompt 模板；`.bob` 整個資料夾目前被 `.gitignore` 排除，見第 5 節說明）。實際操作方式是在 Bob Shell 互動環境中輸入對應指令並帶入參數：

### 步驟 0：文件轉換（Python 腳本，非 Bob 命令）

```bash
python3 ocr_pdf_to_markdown.py <pdf目錄>
python3 convert_to_markdown.py <輸入目錄>
```

輸出帶有 `## 第 X 頁` 頁碼標記的 Markdown，放到 `box/.../markdown/` 底下。

### 步驟 1：第一層蒸餾 — `/process-case`

對應檔案：`.bob/commands/process-case.md`

```
/process-case <case_folder_path> [case_id]
```

- `case_folder_path`：步驟 0 產出的案件 markdown 資料夾（必填）
- `case_id`：手動指定案件編號，例如 `CASE-0002`（選填，留空則自動遞增）

會讀取 6 個 template，逐一生成並寫入：
`overview.md`、`dispute-types.md`、`root-causes.md`、`timeline.md`、`evidence-summary.md`、`lessons-learned.md`，並強制要求每段論述附上 inline citation。

### 步驟 2：第二層蒸餾 — `/multi-case-distillation`

對應檔案：`.bob/commands/multi-case-distillation.md`

```
/multi-case-distillation <case_path>
```

- `case_path`：步驟 1 產出的案件目錄，例如 `wiki/cases/CASE-0001`

讀取該案例的 6 個標準文件，蒸餾出 Case Card、Dispute Type Entry、Actionable Insight Entry，並 **append**（非覆蓋）到 `wiki/meta/cross-case/` 底下對應檔案。具冪等性：同一案例重複執行會詢問是否覆蓋。

**建議：累積至少 3–5 個案例後，再進入第三層。**

### 步驟 3：第三層蒸餾 — `/update-meta-prompt`

對應檔案：`.bob/commands/update-meta-prompt.md`

```
/update-meta-prompt
```

不需參數，會自動讀取所有 Case Cards，統計爭議類型 / 根本原因 / 風險標籤頻率，找出出現在 2+ 案例中的風險模式，產出更新草稿，**等待人工確認後才寫入** `wiki/meta/system/risk-checklist.md`（並自動備份舊版）。案例數少於 3 個時會提示先累積案例。

### 完整跑一輪的順序

```
1. 上傳新案件 PDF
2. python3 ocr_pdf_to_markdown.py ...        # 第零層
3. python3 convert_to_markdown.py ...        # 第零層
4. /process-case <markdown目錄> CASE-000X    # 第一層
5. /multi-case-distillation wiki/cases/CASE-000X  # 第二層
6. 重複 1–5 累積多個案例
7. /update-meta-prompt                       # 第三層，人工確認後生效
```

之後新合約上傳做風險分析時，系統會反向讀取：`risk-checklist.md`（第一層）→ 相關 `dispute-types/*.md` / `actionable-insights/*.md`（第二層）→ 視需要才讀取 Top 3–5 案例細節（第三層），把 context 消耗從 30000+ tokens 壓到 5000–10000 tokens。

### MSA Heatmap（另一條獨立功能線，非三層蒸餾）

`heatmap-style` 模式（見 `.bob/custom_modes.yaml`）搭配 `src/routes/heatmap.js`、`src/services/heatmapService.js` 對合約逐條打分（Red/Amber/Green），輸出 JSON 再渲染成 `heatmap/output/*.html` 報告。評分依據是 `heatmap/template/heatmap_guidance.md`，欄位對應說明見 `heatmap/criteria-mapping.md`。這條路徑走的是 API（`/api/heatmap`），不是透過 `.bob/commands` 的 slash command。


## 5. 專案結構速覽

```
src/
├── server.js              # 唯一可運作的後端入口
├── config/                 # 常數、Bob Shell 路徑解析、heatmap 設定
├── routes/                 # /api/health、/api/chat、/api/cases、/api/heatmap ...
├── services/                # bobService（PTY 呼叫 Bob Shell）、heatmapService、ocrService ...
└── utils/

.bob/commands/               # 三層蒸餾的 slash command 定義（見第 4 節）
heatmap/                      # MSA Heatmap 評分模板、輸出報告、爭議項目對照表
box/                          # Box 雲端案件 wiki 掛載點（真實資料已被 .gitignore 排除）
documentations/, docs/         # 架構文件與知識管理指南
test/                          # API / 上傳流程測試腳本
```

延伸閱讀：

- `documentations/LLM_WIKI_FLOW.md`：三層蒸餾完整流程與 Context 消耗數據
- `CROSS_CASE_DISTILLATION_ARCHITECTURE.md`：跨案例蒸餾架構設計理念
- `docs/KNOWLEDGE_GUIDELINES.md` / `docs/README.md`：inline citation 規範與知識庫寫作規範
- `DEPLOYMENT_GUIDE.md`：容器化部署細節（分享前記得先做第 5.4 節的路徑清理）
