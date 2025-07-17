# 📄 n8n Workflow 文件：GitHub Issue 關閉自動同步 Sentry 狀態

## 🎯 目的
本 workflow 監聽 GitHub Issue 被關閉的事件，並自動同步對應 Sentry Issue 狀態為 resolved。此流程可確保 Sentry 與 GitHub 的錯誤追蹤狀態一致，減少人工同步的負擔。

---

## 🧭 Workflow 流程概述

1. [Webhook - Issue Closed by GitHub]（接收 GitHub 關閉 issue 事件）⮕ 
2. [解析 Sentry API 連結] ⮕ 
3. [呼叫 Sentry API 將 issue 標記為 resolved]

![Sentry 流程圖](https://github.com/katlyn220915/workflows/blob/main/assets/issue-closed-by-github.jpg)

---

## 🧩 Workflow Node 詳解

### 🔹 1. Webhook — `Issue closed by github`

- **Node 名稱**：`Issue closed by github`
- **用途**：接收來自 GitHub 的 webhook，當 issue 被關閉時觸發
- **觸發條件**：GitHub 設定 webhook，指向 n8n instance 的 `/webhook/close-issue` 路徑，事件類型為 `issues` 且 action 為 `closed`
- **必要條件**：
  - GitHub repo 已設定 webhook，指向 n8n instance
  - n8n instance 可公開對外接收 POST 請求
- **輸出**：包含被關閉的 GitHub issue 內容（`body.issue.body`）

---

### 🔹 2. Code — `Get sentry issue url from Github issue body`

- **Node 名稱**：`Get sentry issue url from Github issue body`
- **用途**：從 GitHub issue body 解析出 Sentry Issue API 連結 (從 new-issue-created-workflow 帶到 github issue body 的資訊 )
- **執行邏輯**：
  - 以正則表達式從 issue body 內容中擷取 `**Sentry Issue API:** <url>` 這段的 url
- **輸出欄位**：
  - `sentryApiUrl`：Sentry Issue API 的完整網址

---

### 🔹 3. HTTP Request — `Resolve the sentry issue`

- **Node 名稱**：`Resolve the sentry issue`
- **用途**：呼叫 Sentry API，將對應 issue 狀態設為 resolved
- **執行邏輯**：
  - 使用 HTTP PUT 請求，帶入 Bearer Token 認證
  - Body 參數：`{ "status": "resolved" }`，參考 [Sentry Doc - update issue](https://docs.sentry.io/api/events/update-an-issue/)
- **必要條件**：
  - n8n 中已設定好 Sentry API 的 Bearer Token 憑證（Credential 名稱如 `"Bearer Auth account"`）

---

## 🔒 認證與安全考量

| 整合對象 | 認證方式            | 說明                                         |
|----------|---------------------|----------------------------------------------|
| GitHub   | Webhook             | 需設定 webhook 指向 n8n instance             |
| Sentry   | Bearer Token        | 儲存在 n8n Credential 名為 `"Bearer Auth account"` |

---

## 🧪 測試方式

1. 在 GitHub 測試關閉一個已連結 Sentry 的 issue。
2. 確認 n8n workflow 正確執行，並於 Sentry 對應 issue 狀態變為 resolved。
3. 可於 n8n workflow 執行紀錄中檢查流程是否順利。

---

## 📌 補充說明

- GitHub issue body 需包含 `**Sentry Issue API:** <url>` 格式，否則無法自動同步。
- 若需支援多個 Sentry 專案，可於 Code node 增加對應邏輯。
