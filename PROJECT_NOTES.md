# Wei 財務儀表板 — 專案說明文件

> 給 Claude 看的：這是一個單一 HTML 檔案的個人財務工具。每次新對話時，請把 `index.html` 和這份文件一起上傳，我就能完整掌握專案脈絡。

---

## 專案概覽

- **檔案**：`index.html`（單一檔案，全部 CSS / JS 內嵌，約 3000+ 行）
- **部署**：GitHub Pages → `https://brianiiii456456-debug.github.io/wei-finance/`
- **原始碼**：`https://github.com/brianiiii456456-debug/wei-finance`
- **資料同步**：GitHub Gist（secret），localStorage key 為 `wei_finance_v1`
- **語言**：繁體中文介面

---

## 分頁結構

| 快捷鍵 | 分頁 | 說明 |
|--------|------|------|
| `1` | 儀表板 | 總覽：銀行帳戶、信用卡已刷、可用額度、固定支出、訂閱費用 |
| `2` | 資產 | 銀行帳戶餘額詳細列表 |
| `3` | 訂閱 | 所有訂閱項目（月費/年費，支援多幣別） |
| `4` | 信用卡 | 信用卡已刷金額、額度、可用額度 |
| `5` | ✏ 編輯 | 編輯所有資料（銀行帳戶、固定支出、訂閱、設定） |

編輯分頁的子分頁快捷鍵：`Q`（銀行帳戶）、`W`（固定支出）、`E`（訂閱）、`R`（設定）

---

## 資料結構（`state` 物件）

```js
state = {
  banks: [{ id, name, balance, currency, note }],
  expenses: [{ id, name, amount, note }],
  subs: [{ id, name, amount, currency, cycle, expires, note, extra }],
  cards: [{ id, name, billed, limit, available, note, debitDay, debitBank }],
  settings: { baseCurrency, showRates },
  bankOrder: [...],   // 使用者自訂銀行區塊排序（drag-and-drop）
  history: [...],     // undo 堆疊
}
```

- `currency`：`'TWD'`、`'JPY'`、`'KRW'` 等
- `cycle`：`'monthly'` 或 `'yearly'`
- `extra`：訂閱的匯率修正係數（預設 1.015）
- `debitDay`：信用卡扣帳日（數字或 null）
- `debitBank`：信用卡對應的扣款銀行名稱

---

## 雲端同步機制

- **儲存位置**：GitHub Gist（secret）中的 `wei_finance_data.json`
- **localStorage key（同步設定）**：`wei_sync_cfg`，儲存 `{ token, gistId }`
- **localStorage key（資料）**：`wei_finance_v1`
- **自動推送**：每次 `saveState()` 後延遲 800ms debounce 推送到 Gist
- **自動拉取**：頁面載入後 400ms 自動從 Gist 拉取最新資料
- **連線設定 URL 格式**：`?sync=BASE64(token:gistId)`（query string，非 hash，手機 Safari 較相容）

### 重要的 TDZ 修復
`const SYNC_KEY` 和 `let _syncCfg` 等 sync 相關變數定義在 script 中段，因此 sync 初始化的呼叫（`loadSyncCfg()`、`parseHashSync()`、`pullFromGist()`）**必須放在 script 最末尾**（`render()` 和 `fetchRates()` 之後），否則 `const`/`let` TDZ 會導致靜默錯誤、連線設定無法讀取。

---

## 鍵盤操作設計

### 全域快捷鍵
- `1`–`5`：切換主分頁
- `Q`、`W`、`E`、`R`（在編輯分頁）：切換子分頁
- `Z`（macOS: `Cmd+Z`）：Undo
- `Escape`：關閉 modal / 取消編輯

### 在編輯分頁輸入欄位中
- `Tab` / `Shift+Tab`：移到下一個 / 上一個輸入欄位（**循環**，不跳出）
- `Enter`：移到下一個輸入欄位
- `↓` / `↑`：移到下一個 / 上一個輸入欄位
- **不會跳到按鈕**，游標只在 input / select / textarea 之間移動
- 當游標在最上方時按 `↑` 或 `Shift+Tab`，會循環到最下方的欄位

### 實作細節（`focusAdjacentInput`）
- 收集當前 active tab 內所有可見的 input/select/textarea
- 用 `_pendingFocusIdx` 儲存目標 index，在 `render()` 重建 DOM 後恢復焦點
- `idx < 0`（焦點不在任何輸入框時）直接 return，不跳到第一格

---

## 拖移排序（銀行區塊）

- **拖移對象**：整個「銀行區塊」（含銀行名稱 + 旗下所有信用卡）
- **觸發方式**：必須從區塊標題列的 `⠿` 拖把圖示開始拖
- **實作方式**：`mousedown` 事件追蹤是否從 `.drag-handle` 開始，若否則在 `dragstart` 中 `preventDefault()`
- **排序儲存**：`state.bankOrder` 陣列，`saveState()` 後同步到 Gist

---

## 多幣別匯率

- 自動從公開匯率 API 取得（`fetchRates()`，頁面載入時執行）
- 支援 TWD、JPY、KRW、USD 等
- 訂閱費用顯示時自動換算為台幣（`baseCurrency`，預設 TWD）
- `extra` 欄位可設定個別訂閱的匯率修正係數

---

## 使用者偏好與設計原則

1. **介面語言**：繁體中文
2. **色調**：深色模式（dark mode）為主
3. **操作要直觀**：鍵盤導航要符合直覺，不要有意外跳格
4. **單一檔案**：所有功能都在 `index.html` 裡，不拆分檔案
5. **資料隱私**：Gist 用 secret（不公開），GitHub Pages 本身是公開 URL 但無登入保護（靠 URL 隱蔽）
6. **同步速度**：800ms debounce 推送（比原本 3 秒快）

---

## 常見修改流程

1. 使用者把 `index.html` + 這份 `PROJECT_NOTES.md` 上傳給 Claude
2. 說明想要的修改
3. Claude 修改完輸出新的 `index.html`
4. 使用者下載後上傳到 GitHub repo（替換舊的 `index.html`）
5. GitHub Pages 自動更新（約 1–2 分鐘）

---

## GitHub 資訊

- **Repo**：`brianiiii456456-debug/wei-finance`
- **Branch**：`main`
- **Pages URL**：`https://brianiiii456456-debug.github.io/wei-finance/`
- **部署設定**：Settings → Pages → Source: Deploy from branch `main` / root `/`
