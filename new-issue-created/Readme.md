# 📄 n8n Workflow 文件：Sentry → GitHub 自動錯誤通報流程與 Slack 通知流程

## 🎯 目的
本 workflow 自動處理來自 **Sentry** 的錯誤通報，並轉換為 **GitHub Issue** 以及通知到 **Slack** 頻道，以利錯誤追蹤與修復。

---

## 🧭 Workflow 流程概述

1. [Webhook - Issue Created from Sentry] (接收 Sentry 通報) ⮕ 
2. [Extract Key Fields]（擷取關鍵欄位） ⮕ 
3. [Format & Organize Data]（格式化與整理資料）⮕ 
4. [Create GitHub Issue]（建立 GitHub issue）& [Send message to slack channel (trigger slack workflow)]（同步通知 Slack）

![Sentry 流程圖](https://github.com/katlyn220915/workflows/blob/main/assets/new-issue-created-workflow.jpg)

---

## 🧩 Workflow Node 詳解

### 🔹 1. Webhook — `Issue Created from Sentry`

- **Node 名稱**：`Issue Created from Sentry`
- **用途**：接收來自 Sentry 的 webhook 資料
- **觸發條件**：Sentry Project 中設定 Webhook URL，例如：`[POST] https://<n8n-host>/webhook/new-issue`（sentry 只使用 POST 觸發 webhook）
- **必要條件**：
  - Sentry 已經設定好 custom integration
  - 已在 Sentry 中設定該 Webhook（事件格式為 V2 JSON）
  - n8n instance 可公開對外接收 POST 請求
- **輸出**：原始錯誤事件 JSON，包含 `body.data.event` 結構

---

### 🔹 2. Set — `Extract Key Fields`

- **Node 名稱**：`Extract Key Fields`
- **用途**：從原始 Sentry webhook 收集的資料中擷取重要欄位
- **輸出欄位**：
  - `title`
  - `tags`
  - `environment`
  - `extra`
  - `issue_url`
  - `issue_link`
  - `issue_id`
- **技術細節**：
- 使用 `mode: raw` 搭配 Handlebars 模板語法
- 欄位來源為 `body.data.event`

---

### 🔹 3. Code — `Format & Organize Data`

- **Node 名稱**：`Format & Organize Data`
- **用途**：將 `Filter raw fields` 輸出進一步格式化，萃取並整理欄位，用於建立 GitHub issue
- **執行邏輯**：
  - 將 `tags` 陣列轉為物件，取得 API metadata（method、status、url、site、path 等）
  - 從 `extra` 中解析 `requestConfig` / `responseData`
  - 整合為統一格式的物件

- **輸出欄位**：
  - `title`
  - `environment`
  - `method`
  - `apiPath`
  - `apiStatus`
  - `site`
  - `web_url`
  - `requestConfig`
  - `responseData`
  - `issue_id`
  - `issue_link`
  - `issue_url`

- **必要條件**：
  - `tags` 為 `[ [key, value], ... ]` 格式（Sentry 預設格式）
  - `extra` 中包含 `requestConfig` 和 `responseData` 欄位，否則 fallback 為 `"UNKNOWN"`

---

### 🔹 4. GitHub — `Create GitHub Issue`

- **Node 名稱**：`Create GitHub Issue`
- **用途**：在指定 repo 中建立 GitHub issue
- **目標 Repo**：
  - `owner`: `xxtechec`
  - `repository`: `web-core123-vendor-frontend`
- **Issue 標題**：  
  `{{ title }}`
- **Issue 內容（body）包含**：
  - Sentry issue ID、link、API 資訊
  - API Request / Response（以 code block 呈現）
- **labels**：`sentry`
- **assignee**：`anyone`

- **必要條件**：
  - n8n 中設定好 GitHub 認證憑證（名為 `"GitHub account"`）
  - 個人或 org 的 personal access token 需具備 `repo` 權限（可建立 issue）

---

### 🔹 5. HTTP Request — `Send message to slack channel (trigger slack workflow)`

- **Node 名稱**：`Send message to slack channel (trigger slack workflow)`
- **用途**：將 Sentry 新 issue 事件資訊同步通知到 Slack 頻道，觸發 Slack workflow
- **執行邏輯**：
  - 以 HTTP POST 請求，將 API method、標題、issue 連結、狀態碼、site、webUrl、apiPath、environment、responseData 等欄位傳送到指定 Slack workflow webhook
- **必要條件**：
  - 需有正確的 Slack workflow webhook URL
- **輸入來源**：來自 `Format & Organize Data` node 的輸出
- **POST body 範例**：

```json
{
  "apiMethod": "{{ $json.method }}",
  "taskTitle": "{{ $json.title }}",
  "issueUrl": "{{ $json.issue_link }}",
  "apiStatusCode": "{{ $json.apiStatus }}",
  "site": "{{ $json.site }}",
  "webUrl": "{{ $json.web_url }}",
  "X-Trace-Id": "還沒有值",
  "apiPath": "{{ $json.apiPath }}",
  "environment": "{{ $json.environment }}",
  "responseData": "{{ JSON.stringify($json.responseData) }}"
}
```

---

## 🔒 第三方資源整合

| 整合對象 | 認證方式            | 說明                                         |
|----------|---------------------|----------------------------------------------|
| GitHub   | Personal Access Token | 儲存在 n8n Credential 名為 `"GitHub account"` |
| Sentry   | 無需認證（Webhook）   | 透過 POST 傳送錯誤資料至 n8n 公開 webhook URL |
| Slack    | Webhook URL         | 需於 Slack workflow 取得並設定 webhook URL    |

---

## 🧪 測試方式

1. 先有一份 sentry 格式的 mock data
2. 在 Sentry 手動觸發監聽 issue，並使用 postman 測試 webhook 傳送是否正常。
3. 確認 n8n workflow 正確執行。
4. 到 GitHub 對應 repo 中檢查是否成功建立新的 issue。
5. 檢查 Slack 頻道是否收到通知。

---

## ☀️ 成果

![Result](https://github.com/katlyn220915/workflows/blob/main/assets/result.jpg)

---

## 📌 補充說明

- 若要支援多個 repo / 負責人，可在 Code node 中建立 `site → repo/assignee` 對應表
- Slack workflow 可根據需求自訂通知格式與自動化流程


