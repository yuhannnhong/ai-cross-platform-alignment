# ai-cross-platform-alignment

讓同一個 AI 人格在不同平台與模型之間搬家，並保持身分、記憶與決策一致的實作筆記。

核心原則：**搬的是宿主與模型，不是重新建立人格。** 平台內建記憶、匯入的歷史對話與當下的對話框都只是輔助；可查詢、可追溯、可持續同步的 Warmup（記憶庫 MCP）才是主要資料來源。

## 文件

| 文件 | 內容 |
| --- | --- |
| [GPT 搬家到 Gemini 實作指南](docs/GPT_to_Gemini_Warmup_Migration_Guide.md) | 資料分層、平台中立核心與宿主設定分離、來源標籤、Warmup MCP 驗收、GPT 歷史匯入、跨平台抽考、本機節點（小機）規範、Gemini 自訂指示模板 |
| [本機改住 Antigravity CLI 實作指南](docs/Antigravity_CLI_Local_Host_Guide.md) | Gemini CLI 不再服務個人帳號後，改用 Antigravity CLI（`agy`）：安裝、登入、資料收集選項、權限模式、接上 Warmup MCP 與 OAuth、背景排程叫醒（headless + JSON schema） |

## 搬家順序

1. 保存原平台的完整匯出（原始 ZIP 冷備份）。
2. 把資料分成 Persona／長期記憶／短期狀態／來源紀錄／歷史對話／程式與規格。
3. 宿主指示分平台保存，人格與記憶保持平台中立。
4. 在新平台接上 Warmup MCP，驗收讀、寫、更新、再讀的完整閉環。
5. 匯入歷史對話作為冷資料，不直接寫入精煉記憶。
6. 本機節點改用新的 CLI 宿主，讓排程能在背景叫醒它。
7. 用同一組問題跨平台抽考：看身分、事實、決策版本與來源是否一致，而不是語氣像不像。

## 不要放進這個 repo

這是公開 repo。以下內容一律不提交：

- API Key、OAuth Token、Cookie、Session Token、雲端金鑰
- MCP 伺服器網址（常含密鑰）
- 原平台匯出 ZIP、私人對話全文
- 正式記憶庫 Dump、含個資的測試資料
- 帳號信箱與其他個人資料
