# GitLab API Contracts: 上傳頁已發布檔案管理清單

**Branch**: `002-upload-file-manager` | **Date**: 2026-04-16  
**Base URL**: `https://fox.25sprout.com`  
**Project ID**: `nbu%2Fpublic-pages`  
**Auth**: `Authorization: Bearer {gl_token}` (sessionStorage `gl_token`)

---

## 1. 列出已發布檔案

### Request

```
GET /api/v4/projects/nbu%2Fpublic-pages/repository/tree
  ?path=public%2F{username}
  &ref=main
  &per_page=100
```

| 參數 | 說明 |
|------|------|
| `path` | URL-encoded 的目錄路徑，如 `public%2Falice` |
| `ref` | 固定為 `main` |
| `per_page` | 固定為 `100`（超出 20 筆顯示上限的安全值） |

### Response — 成功 200

```json
[
  {
    "id": "abc123def456",
    "name": "demo.html",
    "type": "blob",
    "path": "public/alice/demo.html",
    "mode": "100644"
  },
  {
    "id": "789xyz",
    "name": "report.html",
    "type": "blob",
    "path": "public/alice/report.html",
    "mode": "100644"
  }
]
```

### Response — 目錄不存在 404

```json
{ "message": "404 Tree Not Found" }
```

前端處理：視同空清單，顯示「尚無已發布的頁面」。

### 前端過濾規則

```javascript
const files = response
  .filter(item => item.type === 'blob' && item.name.endsWith('.html'))
  .slice(0, 20)  // 最多取 20 筆
  .map(item => ({
    filename: item.name,
    filePath: item.path,
    encodedFilePath: encodeURIComponent(item.path),
    pageUrl: `${CONFIG.PAGES_BASE_URL}/${currentUser.username}/${item.name}`
  }));
```

---

## 2. 刪除已發布檔案

### Request

```
DELETE /api/v4/projects/nbu%2Fpublic-pages/repository/files/{encoded_file_path}
Content-Type: application/json
Authorization: Bearer {gl_token}
```

`{encoded_file_path}` = `encodeURIComponent("public/{username}/{filename}.html")`  
例：`public%2Falice%2Fdemo.html`

### Request Body

```json
{
  "branch": "main",
  "commit_message": "remove: alice/demo.html"
}
```

### Response — 成功 204

無 body。前端繼續執行 Step 2（觸發 pipeline）。

### Response — 錯誤

| HTTP Status | 說明 | 前端處理 |
|-------------|------|---------|
| 401 | token 過期或無效 | 顯示「登入已過期，請重新登入」並清除 token |
| 403 | 權限不足 | 顯示「刪除失敗：權限不足，請聯繫管理員」 |
| 404 | 檔案不存在（已被其他分頁刪除） | 顯示「檔案已不存在」，從清單移除該列 |
| 5xx | 伺服器錯誤 | 顯示「刪除失敗，請稍後再試」 |

---

## 3. 觸發 Pipeline（刪除後重新部署）

### Request

```
POST /api/v4/projects/nbu%2Fpublic-pages/pipeline
Content-Type: application/json
Authorization: Bearer {gl_token}
```

### Request Body

```json
{
  "ref": "main"
}
```

### Response — 成功 201

```json
{
  "id": 12345,
  "status": "pending",
  "ref": "main",
  "web_url": "https://fox.25sprout.com/nbu/public-pages/-/pipelines/12345"
}
```

前端處理：pipeline 觸發成功後即完成刪除流程，**不**在清單頁輪詢此 pipeline 狀態（超出本功能範圍）。

### Response — 錯誤

| HTTP Status | 說明 | 前端處理 |
|-------------|------|---------|
| 401 | token 過期或無效 | 顯示「登入已過期，請重新登入」並清除 token |
| 403 | 無觸發 pipeline 權限 | 顯示「刪除成功，但無法觸發重新部署，請聯繫管理員」（檔案已刪，列出此警告） |
| 5xx | 伺服器錯誤 | 同上，區分「刪除已完成但重新部署失敗」 |

---

## 操作序列圖

```
用戶確認刪除
    │
    ▼
DELETE /repository/files/{path}
    │
    ├─ 204 OK ──────────────────────────────────────┐
    │                                               ▼
    │                                  POST /pipeline { ref: "main" }
    │                                               │
    │                                  ┌────────────┴──────────────┐
    │                                  ▼ 201 Created               ▼ error
    │                            顯示「已成功刪除」           顯示「刪除成功，
    │                            從 DOM 移除該列             但重新部署失敗」
    │
    └─ error ──► 顯示對應錯誤訊息，清單維持原狀
```
