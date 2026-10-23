# Research: HTML Page Publisher

**Branch**: `001-html-page-publisher` | **Date**: 2026-04-02  
**Phase**: 0 — 技術未知項調查

---

## Decision 1: OAuth 2.0 PKCE 靜態頁面實作

**Decision**: 使用 OAuth 2.0 Authorization Code + PKCE 流程，透過瀏覽器原生 **Web Crypto API**（`SubtleCrypto.digest`）產生 SHA-256 code_challenge，完全在前端執行，不需要後端 server 或 client_secret。

**Rationale**:
- GitLab 自 13.0 起支援 PKCE（`code_challenge_method=S256`）
- Web Crypto API 在所有目標瀏覽器（Chrome 94+、Firefox 93+、Safari 15+）中均已提供
- PKCE 防止 authorization code interception 攻擊（RFC 7636）
- 不需要 client_secret，適合純靜態頁面部署

**實作要點**:
1. 產生 `code_verifier`：43–128 個隨機字元（`crypto.getRandomValues` + base64url 編碼）
2. 計算 `code_challenge`：`base64url(SHA-256(code_verifier))`
3. 授權 URL：`https://fox.25sprout.com/oauth/authorize?client_id=...&redirect_uri=...&response_type=code&scope=api&code_challenge=...&code_challenge_method=S256&state=...`
4. Token 交換：`POST https://fox.25sprout.com/oauth/token`（body: `grant_type=authorization_code`, `code`, `code_verifier`, `client_id`, `redirect_uri`）
   - GitLab 的 `/oauth/token` 端點支援 CORS，無需後端代理
5. `state` 參數：用於防止 CSRF，交換前驗證 state 與 sessionStorage 中儲存的值相符

**Alternatives considered**:
- **Implicit flow**: 已廢棄（RFC 9700 / OAuth 2.1），token 直接暴露於 URL fragment，不建議使用
- **後端 token exchange**: 需要額外 server 基礎設施，違反純靜態頁面限制
- **Personal Access Token 手動輸入**: 繞過 OAuth，需用戶自行管理 token，安全性不佳，UX 差

---

## Decision 2: GitLab Repository Files API（檔案建立 / 覆蓋）

**Decision**: 使用 GitLab REST API 的 **Repository Files** 端點進行檔案操作：
- 先以 `GET` 檢查檔案是否存在 → 決定使用 `POST`（建立）或 `PUT`（更新）
- 檔案內容以 **base64** 編碼傳輸（GitLab API 要求）

**API 端點**:
```
# 檢查檔案是否存在
GET /api/v4/projects/nbu%2Fpublic-pages/repository/files/{file_path}?ref=main

# 建立新檔案（不存在時）
POST /api/v4/projects/nbu%2Fpublic-pages/repository/files/{file_path}
Body: { branch, content (base64), commit_message, encoding: "base64" }

# 更新現有檔案（已存在時）
PUT /api/v4/projects/nbu%2Fpublic-pages/repository/files/{file_path}
Body: { branch, content (base64), commit_message, encoding: "base64", last_commit_id }
```

- `file_path` URL encode 範例：`public%2Fjohn%2Fmypage.html`
- `last_commit_id`（PUT 需要）：從 GET 回應的 `last_commit_id` 欄位取得

**Rationale**:
- 標準 GitLab REST API，文件完整，支援 CORS（Bearer token auth）
- `encoding: "base64"` 確保任意 HTML 內容（含特殊字符）可安全傳輸

**Alternatives considered**:
- **GitLab GraphQL API**: `createCommitPayload` mutation 更複雜，不適合簡單單檔操作
- **isomorphic-git（JS Git library）**: 需引入大型依賴，且需 CORS proxy 支援 Git protocol

---

## Decision 3: Pipeline 觸發與狀態輪詢

**Decision**: 不使用額外觸發機制。commit 至 `main` branch 後，GitLab CI 自動依 `.gitlab-ci.yml` 觸發 pipeline。透過以下流程取得 pipeline ID 並輪詢：

**流程**:
1. Commit 完成後，立即呼叫：
   ```
   GET /api/v4/projects/nbu%2Fpublic-pages/pipelines?ref=main&per_page=1&order_by=id&sort=desc
   ```
   取得最新 pipeline 的 `id`
2. 輪詢狀態（每 8 秒）：
   ```
   GET /api/v4/projects/nbu%2Fpublic-pages/pipelines/{id}
   ```
   回應包含 `status`、`web_url`
3. 終止條件：`status` 為 `success` / `failed` / `canceled`，或累計輪詢超過 5 分鐘（37.5 次）

**已知限制**:
- 若兩名用戶在同一秒 commit，`sort=desc` 查詢可能取到他人的 pipeline。MVP 階段此競爭條件可接受（內部工具、低並發）。後續可改為從 commit API 回應取 SHA，再以 `GET /pipelines?sha={sha}` 精確查詢。

**Alternatives considered**:
- **Pipeline 觸發 token（Trigger tokens）**: 需在前端儲存觸發 token（安全性差），且觸發與原始 commit 脫鉤
- **GitLab GraphQL subscriptions**: 需 WebSocket，靜態頁面無法直接使用

---

## Decision 4: 原始碼結構

**Decision**: 單一自含式檔案 `public/publisher/index.html`，CSS 與 JS 內嵌於 `<style>` 和 `<script>` 標籤。

**Rationale**:
- 零建置步驟：直接 commit，GitLab CI 原樣部署
- 自含式：不依賴 CDN、不引入任何外部資源（避免 CSP 問題）
- 功能複雜度低，JS 程式碼預估 300–500 行，單檔可維護
- 相容 GitLab Pages Artifact 部署模型（`artifacts.paths: [public]`）

**Alternatives considered**:
- **分離 .css / .js 檔案**: 對此功能規模不必要，增加檔案管理複雜度
- **Vue/React SPA + Webpack**: 需建置步驟，引入 node_modules，與靜態頁面設計矛盾

---

## Decision 5: 專案識別

**Decision**: GitLab API 呼叫統一使用 URL-encoded 專案路徑 `nbu%2Fpublic-pages` 作為 `:id`。

**Rationale**:
- GitLab API 自 9.x 起支援以 `namespace%2Fproject` 形式代替數字 ID
- 路徑為固定已知值，無需動態查詢專案 ID
- 前端程式碼只需一個常數 `PROJECT_PATH = 'nbu%2Fpublic-pages'`

---

## Decision 6: Token 儲存策略

**Decision**: 使用 `sessionStorage` 儲存 `access_token`、`token_type`、`username`；使用 `sessionStorage` 儲存 OAuth PKCE 中間狀態（`code_verifier`、`state`）。

**Rationale**:
- `sessionStorage` 於 tab 關閉時自動清除（符合 spec Assumptions）
- 不跨 tab 共享（隔離性）
- 比 `localStorage` 安全（減少 XSS 長期暴露窗口）
- 輕量，無需 cookie 設定

**Alternatives considered**:
- **localStorage**: 持久化過長，提高 XSS 竊取 token 的風險窗口
- **Memory only（module-level variable）**: 頁面重新整理後需重新登入，UX 差
- **Cookie**: 可能被意外送出至第三方；設定 `httpOnly` 需後端，超出範疇

---

## 前置條件確認

| 項目 | 確認狀態 | 備註 |
|------|----------|------|
| GitLab 版本支援 PKCE | ✅ 假設已支援（GitLab 13.0+） | 管理員確認版本 |
| GitLab OAuth Application 已建立 | ⚠️ 管理員準備 | 需提供 `client_id`、設定 redirect URI |
| 所有用戶已加入 `nbu/public-pages` Developer | ⚠️ 管理員準備 | 否則 commit API 回 403 |
| `.gitlab-ci.yml` 已配置部署 `public/` | ✅ 已確認 | 見 `.gitlab-ci.yml` |
| GitLab API CORS 允許瀏覽器跨域存取 | ✅ GitLab 預設支援 | Bearer token 模式 |
