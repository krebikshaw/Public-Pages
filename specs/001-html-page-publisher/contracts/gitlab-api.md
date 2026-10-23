# GitLab API Contracts

**Branch**: `001-html-page-publisher` | **Date**: 2026-04-02  
**GitLab Instance**: `https://fox.25sprout.com`  
**Target Project**: `nbu/public-pages` (URL-encoded: `nbu%2Fpublic-pages`)

> 本文件列出 Publisher 頁面需呼叫的所有 GitLab API 端點。所有請求均為瀏覽器端 CORS 請求（GitLab 預設允許）。

---

## Authentication

所有端點（除 OAuth 流程外）均需附帶：

```
Authorization: Bearer {access_token}
Content-Type: application/json
```

---

## 1. OAuth 2.0 PKCE 授權流程

### 1a. 導向授權頁（瀏覽器 redirect，非 fetch）

```
GET https://fox.25sprout.com/oauth/authorize
```

| 參數 | 必填 | 說明 |
|------|------|------|
| `client_id` | ✅ | OAuth Application 的 client_id |
| `redirect_uri` | ✅ | 必須與 GitLab OAuth App 設定完全吻合，例如 `https://nbu.fox.25sprout.com/public-pages/publisher/` |
| `response_type` | ✅ | 固定值 `code` |
| `scope` | ✅ | `api`（需要 repository write 權限） |
| `state` | ✅ | 隨機字串，防 CSRF |
| `code_challenge` | ✅ | `base64url(SHA-256(code_verifier))` |
| `code_challenge_method` | ✅ | 固定值 `S256` |

**成功回應**：瀏覽器 redirect 至 `redirect_uri?code={auth_code}&state={state}`

**失敗回應**：`redirect_uri?error=access_denied&error_description=...`

---

### 1b. Token 交換

```
POST https://fox.25sprout.com/oauth/token
Content-Type: application/x-www-form-urlencoded
```

**Request Body**:

| 參數 | 必填 | 說明 |
|------|------|------|
| `grant_type` | ✅ | 固定值 `authorization_code` |
| `client_id` | ✅ | OAuth Application 的 client_id |
| `code` | ✅ | 從 redirect URL 取得的 auth_code |
| `redirect_uri` | ✅ | 與步驟 1a 相同 |
| `code_verifier` | ✅ | PKCE code_verifier（原始隨機字串，未經 hash） |

**成功回應** `200 OK`:
```json
{
  "access_token": "string",
  "token_type": "Bearer",
  "refresh_token": "string",
  "scope": "api",
  "created_at": 1234567890
}
```

**失敗回應** `400 Bad Request`:
```json
{
  "error": "invalid_grant",
  "error_description": "..."
}
```

---

## 2. 取得當前用戶資訊

```
GET https://fox.25sprout.com/api/v4/user
```

**成功回應** `200 OK`:
```json
{
  "id": 42,
  "username": "johndoe",
  "name": "John Doe",
  "avatar_url": "https://..."
}
```

**使用時機**：token 交換成功後立即呼叫，取得 `username` 供後續 API 使用。

**錯誤處理**:
| HTTP Status | 處理方式 |
|-------------|----------|
| `401` | 清除 token，顯示「登入已過期，請重新登入」 |

---

## 3. 檢查目標檔案是否已存在

```
GET https://fox.25sprout.com/api/v4/projects/nbu%2Fpublic-pages/repository/files/{file_path}
```

- `file_path`：URL-encoded 路徑，例如 `public%2Fjohndoe%2Fmypage.html`
- Query 參數：`?ref=main`

**成功回應** `200 OK`:
```json
{
  "file_name": "mypage.html",
  "file_path": "public/johndoe/mypage.html",
  "last_commit_id": "abc123def456",
  "content": "base64-encoded-content",
  "encoding": "base64"
}
```

**用途**：取得 `last_commit_id`（PUT 時需要），並決定後續操作為 `POST`（建立）或 `PUT`（更新）。

**錯誤處理**:
| HTTP Status | 意義 | 處理方式 |
|-------------|------|----------|
| `200` | 檔案存在 | 使用 `PUT` 更新，帶入 `last_commit_id` |
| `404` | 檔案不存在 | 使用 `POST` 建立 |
| `401` | token 失效 | 清除 token，顯示「登入已過期」 |
| `403` | 權限不足 | 顯示「權限不足，請聯繫管理員」 |

---

## 4a. 建立新檔案（檔案不存在時）

```
POST https://fox.25sprout.com/api/v4/projects/nbu%2Fpublic-pages/repository/files/{file_path}
```

- `file_path`：URL-encoded，例如 `public%2Fjohndoe%2Fmypage.html`

**Request Body**:
```json
{
  "branch": "main",
  "content": "{base64-encoded HTML content}",
  "encoding": "base64",
  "commit_message": "publish: add johndoe/mypage.html"
}
```

**成功回應** `201 Created`:
```json
{
  "file_path": "public/johndoe/mypage.html",
  "branch": "main"
}
```

**錯誤處理**:
| HTTP Status | 意義 | 處理方式 |
|-------------|------|----------|
| `400` | 請求格式錯誤 | 顯示「發布失敗：請求格式錯誤」 |
| `401` | token 失效 | 清除 token，顯示「登入已過期」 |
| `403` | 無 Developer 權限 | 顯示「權限不足，請聯繫管理員開通發布權限」 |
| `422` | 檔案已存在（競爭條件）| 改用 PUT 重試 |

---

## 4b. 更新現有檔案（檔案已存在時）

```
PUT https://fox.25sprout.com/api/v4/projects/nbu%2Fpublic-pages/repository/files/{file_path}
```

**Request Body**:
```json
{
  "branch": "main",
  "content": "{base64-encoded HTML content}",
  "encoding": "base64",
  "commit_message": "publish: update johndoe/mypage.html",
  "last_commit_id": "{last_commit_id from GET response}"
}
```

**成功回應** `200 OK`:
```json
{
  "file_path": "public/johndoe/mypage.html",
  "branch": "main"
}
```

**錯誤處理**:
| HTTP Status | 意義 | 處理方式 |
|-------------|------|----------|
| `400` | `last_commit_id` 不符（衝突） | 顯示「檔案已被其他人修改，請重試」 |
| `401` | token 失效 | 清除 token，顯示「登入已過期」 |
| `403` | 無 Developer 權限 | 顯示「權限不足，請聯繫管理員開通發布權限」 |

---

## 5. 查詢最新 Pipeline

```
GET https://fox.25sprout.com/api/v4/projects/nbu%2Fpublic-pages/pipelines
    ?ref=main&per_page=1&order_by=id&sort=desc
```

**使用時機**：commit（POST/PUT）成功後，等待約 1 秒再呼叫此 API。

**成功回應** `200 OK`:
```json
[
  {
    "id": 789,
    "status": "pending",
    "ref": "main",
    "web_url": "https://fox.25sprout.com/nbu/public-pages/-/pipelines/789",
    "created_at": "2026-04-02T10:00:00Z",
    "finished_at": null
  }
]
```

**注意**：若陣列為空，表示 CI/CD 尚未觸發，稍後重試。

---

## 6. 輪詢 Pipeline 狀態

```
GET https://fox.25sprout.com/api/v4/projects/nbu%2Fpublic-pages/pipelines/{pipeline_id}
```

**輪詢間隔**：每 8 秒  
**逾時上限**：5 分鐘（第 37 次輪詢後停止）

**成功回應** `200 OK`:
```json
{
  "id": 789,
  "status": "running",
  "ref": "main",
  "web_url": "https://fox.25sprout.com/nbu/public-pages/-/pipelines/789",
  "created_at": "2026-04-02T10:00:00Z",
  "finished_at": null
}
```

**終止條件**:
| `status` 值 | 處理方式 |
|-------------|----------|
| `success` | 停止輪詢，顯示「部署成功」+ URL |
| `failed` | 停止輪詢，顯示「部署失敗」+ `web_url` 連結 |
| `canceled` | 停止輪詢，顯示「部署已取消」 |
| `skipped` | 停止輪詢，顯示「部署已略過，請聯繫管理員」|
| 其他（pending/running 等） | 繼續輪詢 |

**錯誤處理**:
| 狀況 | 處理方式 |
|------|----------|
| 網路錯誤 | 顯示「無法取得 pipeline 狀態，重試中…」，繼續輪詢 |
| 逾時（5 分鐘） | 顯示「部署逾時，請至 GitLab 確認或聯繫管理員」，停止輪詢 |
| `404` | 顯示「找不到 pipeline，請聯繫管理員」，停止輪詢 |

---

## 前端常數參考

```javascript
const GITLAB_BASE_URL = 'https://fox.25sprout.com';
const GITLAB_OAUTH_BASE = 'https://fox.25sprout.com/oauth';
const PROJECT_PATH = 'nbu%2Fpublic-pages';
const PAGES_BASE_URL = 'https://nbu.fox.25sprout.com/public-pages';
const PIPELINE_POLL_INTERVAL_MS = 8000;   // 8 秒
const PIPELINE_TIMEOUT_MS = 300000;       // 5 分鐘
const PUBLIC_DIR_PREFIX = 'public';       // commit 路徑前綴
```
