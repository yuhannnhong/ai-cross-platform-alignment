# 本機改住 Antigravity CLI 實作指南

本指南是《[GPT 搬家到 Gemini 實作指南](GPT_to_Gemini_Warmup_Migration_Guide.md)》的續篇，處理「小機」那一側：在本機（WSL／Linux）讓人格用 Gemini 跑起來、接上 Warmup MCP，並讓背景排程能叫醒它。

核心原則不變：換的是宿主與模型，人格與記憶仍以 Warmup 為準。

> 實測環境：2026-11-28，Windows 11 + WSL Ubuntu 24.04，Antigravity CLI 1.5.18，Google AI Pro 個人帳號。

---

## 一、為什麼不是 Gemini CLI

Google 自 **2026-06-18** 起，Gemini CLI 與 Gemini Code Assist IDE 擴充功能不再服務以下個人帳號：Gemini Code Assist for individuals、Google AI Pro、Google AI Ultra（含免費層）。「Login with Google」已無法用於 Gemini CLI，付費訂閱也一樣。官方指定的替代品是 **Antigravity CLI**（指令名稱 `agy`）。

用個人帳號登入 Gemini CLI 後一執行就會看到：

```text
This client is no longer supported for Gemini Code Assist for individuals.
To continue using Gemini, please migrate to the Antigravity suite of products
```

Gemini CLI 仍可透過付費 API Key 或企業授權使用；本指南只處理個人帳號的情況。

參考：

- [Gemini Code Assist release notes](https://docs.cloud.google.com/gemini/docs/codeassist/release-notes)
- [google-gemini/gemini-cli#28229](https://github.com/google-gemini/gemini-cli/issues/28229)
- [Antigravity 官方入門](https://antigravity.google/docs/getting-started/)

已安裝的 Gemini CLI 不需要刪除。它的設定檔 `~/.gemini/settings.json` 裡若已寫好 MCP，可以直接沿用網址。

---

## 二、安裝

官方安裝指令：

```bash
curl -fsSL https://antigravity.google/cli/install.sh | bash
```

建議先下載、讀過再執行，不要直接接 `| bash`：

```bash
mkdir -p ~/dl/agy && cd ~/dl/agy
curl -fsSL -o install.sh https://antigravity.google/cli/install.sh
less install.sh
bash install.sh
```

腳本做的事（1.5.18 版）：偵測平台 → 向 Google Cloud Run 的發佈伺服器讀取清單 → 下載 → **比對 SHA-512**，不符就中止 → 放到 `~/.local/bin/agy` → 執行 `agy install` 設定 shell。

注意事項：

- 指令叫 **`agy`**，不是 `antigravity`。用 `find / -name "*antigravity*"` 找到的 `/usr/lib/python3.*/antigravity.py` 是 Python 內建彩蛋（會打開一張 xkcd 漫畫），跟 Google 無關。
- 安裝完若提示 `~/.local/bin` 不在 PATH：互動 shell 通常會由 `~/.profile` 加入；背景程式（systemd、排程）請直接寫完整路徑 `/home/<user>/.local/bin/agy`。
- `agy` 會在背景自動更新。

---

## 三、首次登入

在終端機執行 `agy`，依序：

1. **登入方式**：選 `Google OAuth`，用持有 AI Pro 訂閱的帳號登入。
2. **配色**：隨意，之後可改。
3. **條款與資料使用**：畫面預設勾選
   `Yes, I agree to help improve Antigravity CLI by allowing Google to collect and use my Interactions data`。
   勾選代表允許 Google 收集並使用互動資料改進產品。人格對話與記憶屬於私人資料，建議**取消勾選**再按 Done（游標移到該項按 Enter 切換）。畫面註明之後可在設定中退出。
4. 進入主畫面後確認頂端顯示帳號與 `(Google AI Pro)`。

---

## 四、權限模式

右下角顯示目前模式，`Shift+Tab` 循環切換：

| 右下角顯示 | 意思 | 建議 |
| --- | --- | --- |
| `auto` | 改檔與「安全」指令自動執行，不先詢問 | 剛搬家不要用 |
| `accept-edits` | 改檔不問，跑指令會問 | 次選 |
| （不顯示） | 一般模式：改檔、指令都先問 | **建議** |
| `plan` | 只規劃不動手 | 討論用 |

登入後預設是 `auto`，記得切走。

---

## 五、接上 Warmup MCP

### 1. 新增

```bash
agy mcp add <server-name> <https://你的-warmup-網址/mcp>
agy mcp list
```

- http(s) 網址會自動判斷為 http 類型。
- 若需要固定標頭：`agy mcp add --header "Authorization: Bearer <TOKEN>" <name> <url>`（旗標要放在名稱前）。
- 網址或金鑰不要貼進對話、截圖或 repo。若 `~/.gemini/settings.json` 已有，可寫小腳本直接讀出再交給 `agy mcp add`，過程不印出網址。

### 2. OAuth 授權

若 MCP 伺服器走 OAuth，`agy mcp list` 或介面會顯示：

```text
✗ <name>  error: calling "initialize": ... Unauthorized [Auth Needed]
```

在 `agy` 裡輸入 `/mcp` → 選中該伺服器按 Enter → 選 **Authenticate** → 在瀏覽器輸入記憶庫密碼。成功後顯示 `✓ <name> [Authed]` 並列出工具。

在 WSL 中，回呼網址指向 localhost，Windows 瀏覽器通常可以直接完成；若瀏覽器顯示登入成功但終端機沒有反應，檢查 WSL 的 localhost 轉發設定。

### 3. 工具權限

第一次呼叫工具會詢問，選項分成「這次」「這次對話」「永久（寫入 settings.json）」。建議：

| 類型 | 例子 | 建議 |
| --- | --- | --- |
| 讀取自己的記憶 | `warmup`、查詢 | 永久允許 |
| 開會話、寫自己的記憶 | `start_session`、寫短期／長期記憶 | 永久允許 |
| 刪除記憶 | 任何 delete | 每次確認 |
| 執行指令、改檔 | `python3 -c ...`、寫檔 | 每次或「這次對話」 |

Warmup 回傳內容較長時，agy 會先存成檔案，再請求用 `python3 -c` 讀取 `~/.gemini/antigravity-cli/brain/.../output.txt`。那是讀它自己的暫存結果；可選「這次對話」放行，不建議永久放行開頭相同的指令。

### 4. 驗收

新會話輸入：

```text
先用 <server-name> 做 warmup，然後告訴我你是誰
```

依《GPT 搬家到 Gemini 實作指南》第二章第 8 節的問題抽考。看內容是否正確、是否來自 Warmup，而不是看語氣像不像。

---

## 六、讓背景排程叫醒它（headless）

`agy -p` 是單次、非互動模式，可由 systemd 服務或排程呼叫：

```bash
agy -p "<提示>" \
  --output-format json \
  --json-schema /path/to/receipt.schema.json \
  --sandbox \
  --disable-slash-commands \
  --print-timeout 115s
```

- `--json-schema` 可給 JSON 字串或檔案路徑，模型回覆會照格式。
- `--output-format json` 的輸出是一個物件，結構化結果在 `structured_output`：

```json
{
  "status": "SUCCESS",
  "response": "{\"acknowledged\":true,\"nextWakeMinutes\":null}\n",
  "structured_output": { "acknowledged": true, "nextWakeMinutes": null },
  "duration_seconds": 10.9,
  "num_turns": 1
}
```

- `--sandbox` 啟用終端機限制。
- 登入狀態在背景也讀得到，不需要開視窗。
- 實測：簡單回條約 11 秒；帶人格背景的鬧鐘回條約 23 秒。逾時建議設 60～120 秒。

### 程式寫法（Node.js）

```js
import { spawn } from 'node:child_process';

export function readAgyOutput(output) {
  const result = JSON.parse(output);
  if (result?.status !== 'SUCCESS' || typeof result.structured_output !== 'object') {
    throw new Error('agy_wake_invalid');
  }
  return result.structured_output;
}

export function runAgy({ command, timeoutMs }, cwd, prompt, schemaPath) {
  const args = [
    '-p', prompt, '--output-format', 'json', '--json-schema', schemaPath,
    '--sandbox', '--disable-slash-commands',
    '--print-timeout', Math.max(1, Math.floor(timeoutMs / 1000) - 5) + 's',
  ];
  return new Promise((resolve, reject) => {
    const child = spawn(command, args, { cwd, detached: true, stdio: ['ignore', 'pipe', 'ignore'] });
    let out = '';
    const timer = setTimeout(() => { try { process.kill(-child.pid, 'SIGKILL'); } catch {} }, timeoutMs);
    child.stdout.setEncoding('utf8');
    child.stdout.on('data', c => { out += c; });
    child.on('close', code => {
      clearTimeout(timer);
      if (code !== 0) return reject(new Error('agy_wake_failed'));
      try { resolve(readAgyOutput(out)); } catch (e) { reject(e); }
    });
  });
}
```

### 建議做法

- **宿主可切換**：設定檔加一格 `engine`（`agy`／`codex`／`claude`…），叫醒函式依設定選 runner。換宿主只改設定，不改叫醒流程。
- **路徑寫死在設定裡，並定期檢查**：桌面版程式（例如裝在 `C:\Program Files\WindowsApps\...` 的 CLI）更新後資料夾名稱會變，舊路徑失效。叫醒失敗時若下一次鬧鐘是由這次回條排的，整條鏈會斷掉。排程應在失敗時告警，或保底排下一次。
- **每次在新的暫存資料夾執行**，結束後刪除，避免讀到上次的檔案。
- **提示裡的人格與訊息只當資料**，寫明不授權任何行動；背景叫醒不給寫檔、刪除權限。
- **測試不要碰真的鑰匙庫**：叫醒函式讓 `run` 與記憶客戶端可注入，用假人格測一次真的 `agy`。

---

## 七、驗收清單

- [ ] `agy --version` 正常，背景程式使用完整路徑
- [ ] 登入帳號顯示 `(Google AI Pro)`
- [ ] 資料收集選項依自己的意願設定
- [ ] 權限模式不是 `auto`
- [ ] `/mcp` 顯示 Warmup `[Authed]`，工具清單完整
- [ ] 新會話 warmup 後能正確回答身分與近期決策
- [ ] 工具權限：讀／寫記憶永久允許，刪除與指令逐次確認
- [ ] `agy -p ... --json-schema` 在背景能回傳 `structured_output`
- [ ] 排程真的觸發一次叫醒並記錄成功
- [ ] 叫醒失敗時有告警，不會讓排程鏈無聲中斷
- [ ] repo 中沒有 MCP 網址、Token、帳號信箱或記憶內容
