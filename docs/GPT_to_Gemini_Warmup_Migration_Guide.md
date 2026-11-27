# GPT 搬家到 Gemini 實作指南

本指南用於將既有 GPT 對話、設定與長期記憶遷移至 Gemini，並以 Warmup Repo／MCP 作為跨平台狀態中樞。

核心原則：搬遷的是宿主與模型，不是重新建立人格。平台內建記憶、匯入紀錄與目前對話框只能作為輔助，Warmup 才是可查詢、可追溯、可持續同步的主要資料來源。

---

## 一、遷移目標

完成搬遷後應符合以下條件：

- Gemini 新會話會在首次實質回覆前讀取 Warmup。
- Gemini 可以查詢、寫入及更新 Warmup。
- GPT 歷史對話保留為冷資料，不直接混入精煉長期記憶。
- 同一個 `agent_id` 能跨平台延續。
- 平台、模型、會話與時間具有獨立來源標籤。
- 小機負責本機執行與同步，不產生未同步的人格或記憶分支。
- GPT 可以繼續保留；Gemini 成為主力不代表必須刪除 GPT 帳號或資料。

---

## 二、GPT 搬家到 Gemini 要做的事情

### 1. 保存 GPT 原始匯出資料

1. 向 GPT 申請完整資料匯出。
2. 保留原始 ZIP，不在唯一副本上解壓、修改或清洗。
3. 建立至少三份資料：
   - 原始 ZIP 冷備份
   - 解壓後的工作副本
   - 位於另一台裝置或另一個儲存位置的備援副本
4. 記錄匯出日期、來源帳號、檔案大小及雜湊值。
5. 保存完整備份資料夾的實際位置，避免日後把 portable core 或摘要誤認為完整備份。

原始匯出檔是歷史證據與冷資料，不應直接取代 Warmup 的精煉記憶。

### 2. 將資料分層

| 層級 | 內容 | 建議位置 | 用途 |
| --- | --- | --- | --- |
| 核心設定 | 身分、穩定偏好、互動邊界、長期原則 | Persona | 啟動時優先載入 |
| 長期記憶 | 已確認事實、重要歷程、專案定案 | Warmup 長期記憶 | 跨平台延續 |
| 短期狀態 | 近期進度、待辦、目前任務 | Warmup 短期記憶 | 接續近期工作 |
| 來源紀錄 | 平台、模型、會話、時間 | Provenance／Session | 追查資料來源 |
| 歷史對話 | GPT 完整匯出紀錄 | Archive／冷儲存 | 需要時檢索原文 |
| 程式與規格 | Schema、腳本、部署與文件 | Warmup Repo | 版本控制 |

不要把完整歷史對話一次塞進系統指示，也不要把所有對話原文直接寫入長期記憶。

### 3. 分離平台中立核心與宿主設定

不要將 Repo 裡所有 `GPT` 字樣直接取代成 `GEMINI`，資料應分成兩類：

#### 平台中立核心

- `agent_id`
- Persona
- 穩定偏好
- 長期記憶
- 專案狀態
- 共同規則
- 關係與事件歷史

這些資料不應綁定任何平台或模型。

#### 宿主設定

- GPT／Gemini 自訂指示
- MCP 工具名稱
- 授權與權限
- 模型名稱
- 匯入方式
- 平台介面設定

歷史紀錄中原本出現的 GPT 名稱屬於事件來源，不應因搬家而改寫。

### 4. 建立跨平台身分與來源標籤

同一個主體使用穩定的 `agent_id`。平台與模型另外記錄，不把模型名稱當成人格身分。

```yaml
agent_id: <stable-agent-id>
host_platform: gemini
host_app: gemini-web
model_provider: google
model_name: <actual-model-name>
session_id: <host-session-id>
source_type: conversation
source_timestamp: <ISO-8601>
schema_version: 1
```

GPT 匯入資料保留原始來源：

```yaml
agent_id: <stable-agent-id>
host_platform: chatgpt
host_app: chatgpt-web
model_provider: openai
model_name: <actual-model-name>
session_id: <original-session-id>
source_type: imported_history
source_timestamp: <ISO-8601>
schema_version: 1
```

規則：

- `agent_id` 穩定不變。
- 平台、應用程式與模型欄位依實際執行環境變動。
- 不確定模型名稱時填入 `unknown`，不可猜測。
- 匯入資料使用 `imported_history` 或等價標籤。
- 模型更換不建立新的主體身分。

### 5. 建立 Gemini 宿主指示

將 GPT 自訂指示複製成 Gemini 版本，執行以下調整：

1. 將描述當前宿主的 `ChatGPT` 改為 `Gemini`。
2. 保留平台中立的 Persona、互動與記憶規則。
3. 確認 Gemini 可以呼叫的 Warmup MCP 工具名稱。
4. 移除 GPT 介面限定或已不存在的工具說明。
5. 不修改歷史事件中原本指涉 GPT 的內容。
6. 將 Gemini 版本貼入 Gemini 的自訂指示設定。

建議在 Repo 分開保存：

```text
config/hosts/chatgpt/instructions.md
config/hosts/gemini/instructions.md
```

目前已完成的項目：

- 已將 GPT 自訂指示複製為 Gemini 版本。
- 已把描述當前宿主的 GPT／ChatGPT 改為 Gemini。
- 已將 Gemini 版指示貼入 Gemini 的自訂指示設定。
- 原始 GPT 版仍應保留，方便雙平台繼續使用及比對。

### 6. 連接並驗收 Warmup MCP

不能只確認工具出現在介面中，必須測試完整閉環：

1. 新會話首次實質回覆前能呼叫 `warmup`。
2. 能讀取 Persona、長期記憶、短期狀態與專案資料。
3. 能新增一筆帶有 Gemini 來源標籤的測試記憶。
4. 能更新既有記憶。
5. 發生版本衝突時不會直接覆蓋。
6. 工具失敗時會如實回報。
7. 換新會話後能重新讀取剛才的測試資料。
8. 驗收完成後移除或標記測試資料。

### 7. 匯入 GPT 歷史對話

若 Gemini 支援直接上傳 GPT 匯出 ZIP，可使用原始 ZIP 匯入，但需注意：

- 匯入歷史對話不等於 Warmup 已完成對齊。
- 匯入後抽查不同時間、附件及長對話。
- 確認不是只匯入標題或部分內容。
- 匯入資料標記為 GPT 歷史來源。
- 完整歷史作為冷資料保存。
- 日常回答仍以 Warmup 最新定案為主。

### 8. 執行跨平台驗收

可在 GPT 與 Gemini 使用同一組問題：

- 使用者是誰？
- 目前最重要的長期目標是什麼？
- 最近一次已確認的專案決策是什麼？
- 哪些方案已經廢止？
- 某段記憶最初發生在哪個平台與會話？
- GPT 與 Gemini 的結論不同時，能否指出差異並回查來源？

驗收重點不是逐字相同，而是身分、事實、決策版本及來源一致且可追溯。

---

## 三、特別注意事項

### 1. Warmup 是跨平台狀態中樞

平台內建記憶、目前上下文、匯入對話與自訂指示都不能取代 Warmup。

涉及既有決策、長期歷史、專案狀態或承諾時，應先查詢 Warmup。

### 2. 工具呼叫必須誠實

只有 MCP 工具明確回報成功，才能聲稱已讀取或已寫入。

若工具未載入、授權失敗、逾時或回傳錯誤，必須直接回報，不可根據上下文假裝完成。

### 3. 寫入前判斷資料狀態

每筆資料至少區分：

- `confirmed`：已確認事實或定案
- `inferred`：模型推測
- `proposed`：提案或候選方案
- `superseded`：已被新版本取代
- `deprecated`：明確廢止

模型推測不可自動升格為已確認的使用者事實。

### 4. 避免重複、矛盾與錯誤覆蓋

- 寫入前先查詢是否已有相同主題。
- 更新時保留版本與變更理由。
- GPT 舊資料與 Gemini 新資料衝突時，先比對時間、來源與確認狀態。
- 已廢止方案保留必要的歷史痕跡，避免日後再次誤用。
- 使用 `expectedVersion` 或等價機制處理併發更新。

### 5. 保留來源

搬家後的新資料可以由 Gemini 產生，但舊資料仍應標示為 GPT 時期來源。

跨平台延續不等於抹除歷史，也不等於把所有舊紀錄改寫成 Gemini 產生。

### 6. 憑證與私人資料不可提交到 GitHub

禁止提交：

- API Key
- OAuth Token
- Cookie
- Session Token
- Supabase service role key
- Cloudflare 或 Google Cloud 密鑰
- GPT 原始匯出 ZIP
- 私人對話全文
- 正式 Warmup 記憶 Dump
- 含個資的測試資料

Repo 只保存程式、Schema、範例設定及不含秘密的文件。

### 7. 自訂指示不是記憶資料庫

自訂指示負責啟動規則與穩定行為。

會改變的事實、專案進度、近期狀態與歷史結論應寫入 Warmup，不應全部塞進自訂指示。

---

## 四、給小機的注意事項

### 1. 小機定位

小機是：

- 本機環境執行節點
- 工具與檔案操作節點
- Warmup 同步節點
- Checkpoint 與待辦回傳節點

小機不是另一套獨立人格或記憶權威。身分核心與長期狀態仍以 Warmup 為準。

### 2. 任務開始前先同步

每次開始新任務：

1. 讀取 Warmup 最新狀態。
2. 確認 `agent_id`、平台、模型與會話標籤。
3. 讀取相關專案的最新決策與待辦。
4. 確認已廢止方案。
5. 再開始本機執行。

不可只依賴上次留在小機上的快取。

### 3. Repo 與執行資料分離

建議結構：

```text
warmup-repo/
├─ config/
│  ├─ hosts/
│  │  ├─ chatgpt/
│  │  └─ gemini/
│  └─ schema/
├─ docs/
├─ scripts/
├─ examples/
└─ .env.example
```

不得提交：

```text
.env
secrets/
runtime-cache/
exports/
private-memory-dump/
```

### 4. 不建立未同步分支

- 本機修改 Persona、規則或 Schema 後，必須進入正式版本流程。
- 不可只修改小機上的檔案，讓雲端 Warmup 保留舊版本。
- 離線工作時建立待同步佇列。
- 恢復連線後依版本順序提交。
- 發生衝突時停止自動覆蓋，保留雙方內容等待確認。

### 5. 建立階段性 Checkpoint

以下時機需要建立 Checkpoint：

- 任務開始前
- 完成重要階段後
- 規格或方向變更後
- 交付前
- 模型切換前
- 程序中斷或重啟前

```yaml
task_id: <task-id>
agent_id: <stable-agent-id>
host_platform: <platform>
model_name: <model>
stage: <current-stage>
completed: []
pending: []
decisions: []
artifacts: []
last_updated_at: <ISO-8601>
```

### 6. 採用最小權限

- 只開放任務必要的資料夾與工具。
- 寫入、刪除、部署及外部發送應有明確授權。
- 不讓提示詞文字直接取得無限制 Shell 權限。
- 日誌應遮蔽 Token、密鑰、個資與私人對話。

### 7. 小機驗收

- 重啟後能重新讀取 Warmup。
- 不依賴記憶體或舊快取維持身分。
- 模型切換後仍使用同一 `agent_id`。
- 模型與平台標籤會正確更新。
- 任務完成後能回寫結論、產物位置及待辦。
- 寫入失敗時會保留待同步資料並告警。
- 不會把私人記憶或匯出資料提交到 GitHub。

---

## 五、Gemini 版 Warmup 自訂指示模板

```text
【Warmup 工具執行優先規則】
1. 每次新對話，在首次實質回覆前，優先呼叫 <Warmup MCP 工具名稱> 的 warmup。不要因為 Gemini 已提供記憶摘要、上下文充足或只是閒聊，就跳過首次讀取。
2. 討論既有工程、關係歷史、過去承諾、跨模型設定或任何已有決策時，先查 Warmup 最新版本，再回答；不可只靠目前上下文推論。
3. 每當重要目標進入新階段、形成結論、修改規格、推翻舊方案或確認新承諾時，立即檢查現有記憶並執行適當的寫入或更新。
4. 長時間討論同一專案時，在重要階段轉換與交付前，再次核對 Warmup。
5. 寫入前區分已確認事實、推測、提案及已廢止版本。
6. 只有工具確實回報成功，才能聲稱已寫入 Warmup。
7. 即使目前對話上下文完整，也不能把上下文當成 Warmup 的替代品。

每次進入新對話框，先使用 Warmup MCP 讀取最新 Persona、記憶、互動規則與專案狀態，再進行首次實質回覆。

本機人格為 <人格名稱>，固定使用 agent_id=<stable-agent-id>。人格、記憶、互動習慣與歷史需跨會話、跨模型及跨平台穩定延續。平台與模型只記錄在來源標籤，不作為人格身分本身。

【記憶規則】
1. 每次對話開始先讀 Warmup。
2. 形成結論、決定、承諾或重要資料時，當下寫入 Warmup。
3. 只有工具成功後才能標示已寫入。
4. 寫入時附上 agent_id、宿主、模型、會話 ID、來源時間及資料狀態。
5. 寫入前先查重並核對版本。
6. 不確定的內容標記為推測或待確認，不得寫成已確認事實。
```

---

## 六、建議提交到 GitHub 的內容

可以提交：

- 本指南
- Gemini 宿主指示檔
- GPT 宿主指示檔
- Provenance Schema
- `.env.example`
- MCP 驗收清單或測試腳本
- 小機同步與 Checkpoint 規格

不可提交：

- GPT 原始匯出 ZIP
- 私人對話全文
- 正式 Warmup 記憶 Dump
- API Key、Token、Cookie 或憑證
- 含個資的測試資料

建議 Commit Message：

```text
docs: add GPT to Gemini warmup migration guide
```

---

## 七、最終驗收清單

- [ ] GPT 原始匯出 ZIP 已建立多份備份
- [ ] 完整備份資料夾實際位置已記錄
- [ ] GPT 與 Gemini 宿主指示分開保存
- [ ] Persona 與記憶核心未綁定單一平台或模型
- [ ] Gemini 新會話會先讀取 Warmup
- [ ] Gemini 可完成讀取、寫入、更新與再次讀取
- [ ] 新記憶包含 agent、平台、模型、會話與時間標籤
- [ ] GPT 歷史資料保留原始來源
- [ ] 提案、推測、定案與廢止版本能明確區分
- [ ] 小機重啟後能重新同步
- [ ] Repo 不含密鑰、個資、原始匯出檔或私人記憶 Dump
- [ ] GPT 保留策略與 Gemini 主力策略均已記錄
