# Data Model: HTML Page Publisher

**Branch**: `001-html-page-publisher` | **Date**: 2026-04-02  
**Phase**: 1 — 設計階段

> 本功能無後端資料庫。所有實體為**瀏覽器端 in-memory / sessionStorage 狀態**，頁面關閉後消失。

---

## Entities

### User（登入用戶）

儲存於：`sessionStorage`（token 部分），in-memory（其餘）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `username` | `string` | GitLab username，用於組合 commit 路徑與部署 URL |
| `name` | `string` | GitLab 顯示名稱，用於 UI 顯示 |
| `accessToken` | `string` | OAuth access token，存於 `sessionStorage['gl_token']` |
| `tokenType` | `"Bearer"` | 固定值，用於 API Authorization header |

**驗證規則**:
- `username` 必須為非空字串，且僅含合法 GitLab username 字元（`[a-zA-Z0-9._-]`）
- `accessToken` 必須為非空字串；若 API 回應 401，視為 token 失效，清除並提示重新登入

---

### HTMLPage（待發布頁面）

in-memory，代表單次上傳操作的意圖。

| 欄位 | 型別 | 說明 |
|------|------|------|
| `originalFilename` | `string` | 用戶選取的原始檔名，例如 `mypage.html` |
| `targetFilename` | `string` | 用戶指定的目標檔名（不含 `.html`），例如 `mypage` |
| `targetPath` | `string` | 計算出的完整路徑 `public/{username}/{targetFilename}.html` |
| `encodedContent` | `string` | HTML 檔案內容的 base64 字串（供 GitLab API 使用） |
| `operation` | `"create" \| "update"` | 由 GET 檔案 API 決定：檔案不存在 → `create`；存在 → `update` |
| `lastCommitId` | `string \| null` | 僅 `update` 時需要，從 GET 回應取得 || `fileHash` | `string \| null` | HTML 檔案 base64 內容的 SHA-256 前 2 bytes，轉為 4 字元十六進位字串（如 `a3f2`）；選擇檔案時即時計算；用於組合 `?v={hash}` cache-busting URL 參數 |
**驗證規則**（對應 FR-005）:
- `targetFilename` 僅允許 `[a-zA-Z0-9_-]`，長度 1–100 字元
- 禁止含有 `/`、`\`、`..`、空白字元（路徑穿越防護）
- 檔案大小上限：5 MB（超過則提示「檔案過大」）
- MIME type 驗證：檔案的 `type` 屬性必須為 `text/html`，或副檔名為 `.html` / `.htm`

---

### Pipeline（部署任務）

in-memory + `sessionStorage`（pipeline ID 持久化，供頁面重整後恢復）

| 欄位 | 型別 | 說明 |
|------|------|------|
| `id` | `number` | GitLab pipeline 數字 ID |
| `status` | `PipelineStatus` | 目前狀態（見下方狀態定義）|
| `webUrl` | `string` | GitLab pipeline 詳情頁 URL，供用戶點擊查看完整 log |
| `createdAt` | `string` | ISO 8601 時間字串 |
| `finishedAt` | `string \| null` | 完成時間，進行中為 null |
| `ref` | `string` | 分支名稱，固定為 `main` |

**PipelineStatus 型別**（GitLab 回傳值）:
```
"created" | "waiting_for_resource" | "preparing" | "pending" |
"running" | "success" | "failed" | "canceled" | "skipped"
```

**顯示映射**:
| GitLab status | 用戶看到的顯示 | 輪詢繼續？ |
|---|---|---|
| `created` / `waiting_for_resource` / `preparing` / `pending` | 等待中… | ✅ |
| `running` | 部署中… | ✅ |
| `success` | 部署成功 ✓ | ❌（停止）|
| `failed` | 部署失敗 ✗ | ❌（停止）|
| `canceled` | 部署已取消 | ❌（停止）|
| `skipped` | 部署已略過 | ❌（停止）|


---

### DeploymentURL（部署網址）

純函式計算，無需儲存。

```
deploymentURL(username, filename, hash) =
  "https://nbu.fox.25sprout.com/public-pages/" + username + "/" + filename + ".html"
  + (hash ? "?v=" + hash : "")
```

`hash` 即 `fileHash`，4 字元十六進位字串；若無法計算（如檔案讀取失敗）則省略 `?v=` 整段。

---

### OAuthPKCEState（OAuth 中間狀態）

儲存於 `sessionStorage`，僅在 OAuth redirect 流程期間使用，完成後清除。

| 欄位 | 型別 | 儲存 key | 說明 |
|------|------|----------|------|
| `codeVerifier` | `string` | `gl_pkce_verifier` | 43–128 字元隨機字串 |
| `state` | `string` | `gl_oauth_state` | 隨機字串，用於 CSRF 防護 |
**其他 sessionStorage 鍵**（不屬於 PKCE，但需一併紀錄）：

| 儲存 key | 內容 | 生命週期 |
|------------|------|----------|
| `gl_token` | OAuth access token | 登入後持續，登出時清除 |
| `gl_pipeline_id` | 正在輪詢的 pipeline，數字 ID | pipeline 結束或症時清除 |
| `gl_file_hash` | 對應 `gl_pipeline_id` 的檔案 `fileHash`，之後組合 `?v=` | pipeline 流程結束或症時清除 |
---

## State Machine（App 狀態機）

```
UNAUTHENTICATED
  ├─[點擊登入]──────────────────→ OAUTH_REDIRECT（redirect 至 GitLab 授權頁）
  └─[redirect 回來 + code]──────→ OAUTH_CALLBACK

OAUTH_CALLBACK
  ├─[state 驗證失敗]────────────→ ERROR（顯示「登入失敗：安全驗證錯誤」）
  ├─[token 交換失敗]────────────→ ERROR（顯示「登入失敗：無法取得 token」）
  └─[token 交換成功]────────────→ AUTHENTICATED

AUTHENTICATED
  ├─[點擊登出]──────────────────→ UNAUTHENTICATED（清除 sessionStorage）
  └─[選檔 + 填名 + 點發布]──────→ UPLOADING

UPLOADING
  ├─[前端驗證失敗]──────────────→ AUTHENTICATED（顯示 inline 錯誤，不離開此狀態）
  ├─[API 401]───────────────────→ UNAUTHENTICATED（顯示「登入已過期」）
  ├─[API 403]───────────────────→ AUTHENTICATED（顯示「權限不足，請聯繫管理員」）
  ├─[API 其他錯誤]──────────────→ AUTHENTICATED（顯示錯誤訊息）
  └─[commit 成功]───────────────→ POLLING

POLLING（每 8 秒輪詢）
  ├─[status: success]───────────→ DEPLOYED
  ├─[status: failed/canceled]──→ DEPLOYMENT_FAILED
  ├─[輪詢超過 5 分鐘]──────────→ DEPLOYMENT_TIMEOUT
  └─[網路錯誤]─────────────────→ POLLING（顯示 retry 警告，繼續輪詢）

DEPLOYED          → 顯示成功 + URL（可再次上傳，回到 AUTHENTICATED）
DEPLOYMENT_FAILED → 顯示錯誤訊息 + pipeline 連結（可重試，回到 AUTHENTICATED）
DEPLOYMENT_TIMEOUT→ 顯示逾時提示（可重試，回到 AUTHENTICATED）
```

---

## 前端模組職責（邏輯劃分）

雖然最終為單一 HTML 檔案，JS 邏輯應依職責組織為明確函式群組：

| 模組（函式群）| 職責 |
|--------------|------|
| `Auth` | PKCE 生成、OAuth redirect、token 交換、sessionStorage 讀寫、logout |
| `GitLabAPI` | Fetch 包裝、Authorization header 注入、錯誤映射（401/403/4xx/5xx → 中文訊息）|
| `FileOp` | 檔案讀取（FileReader）、base64 編碼、前端驗證（格式、大小、命名）|
| `PipelinePoller` | 輪詢排程（setInterval）、逾時計時（setTimeout）、狀態轉換觸發 |
| `UI` | DOM 操作、狀態顯示切換、按鈕 disable/enable、錯誤/成功訊息渲染 |
