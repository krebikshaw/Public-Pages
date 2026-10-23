# Quickstart: HTML Page Publisher

**Branch**: `001-html-page-publisher` | **Date**: 2026-04-02

---

## 前置條件

在開始部署前，GitLab 管理員需完成以下準備：

### 1. 建立 GitLab OAuth Application

1. 以管理員身份登入 `https://fox.25sprout.com`
2. 前往 **Admin Area → Applications（或個人 User Settings → Applications）**
3. 建立新 Application：
   - **Name**: `NBU Page Publisher`
   - **Redirect URI**: `https://public-pages-56dd79.fox.25sprout.com/publisher/`
   - **Scopes**: 勾選 `api`
   - **Confidential**: **不勾選**（PKCE 流程不需 client_secret）
4. 記下 **Application ID**（即 `client_id`）

### 2. 將使用者加入 nbu/public-pages 專案

1. 前往 `https://fox.25sprout.com/nbu/public-pages/-/project_members`
2. 將所有需要使用 Publisher 的用戶加入，角色設為 **Developer** 或以上
3. 沒有 Developer 權限的用戶發布時會看到「權限不足」錯誤

### 3. 確認 `.gitlab-ci.yml` 已配置

本 repo 的 `.gitlab-ci.yml` 應已配置部署 `public/` 目錄。確認如下格式存在：

```yaml
pages:
  stage: deploy
  script:
    - echo "Deploying GitLab Pages..."
  artifacts:
    paths:
      - public
  only:
    - main
```

---

## 本機開發設定

### 1. Clone 專案

```bash
git clone git@fox.25sprout.com:nbu/public-pages.git
cd public-pages
```

### 2. 建立 Publisher 頁面

Publisher 頁面位於 `public/publisher/index.html`。需在檔案頂部的設定區塊填入 `client_id`：

```html
<!-- 設定區塊（檔案頂部附近）-->
<script>
  const CONFIG = {
    GITLAB_BASE_URL: 'https://fox.25sprout.com',
    CLIENT_ID: 'YOUR_OAUTH_APP_CLIENT_ID',   // ← 填入步驟 1 取得的 Application ID
    REDIRECT_URI: 'https://public-pages-56dd79.fox.25sprout.com/publisher/', // 正式生產實際値
    PROJECT_PATH: 'nbu%2Fpublic-pages',
    PAGES_BASE_URL: 'https://nbu.fox.25sprout.com/public-pages',
    CDN_WARMUP_MS: 10000, // Pipeline 成功後 CDN 生效倒數秒數
  };
</script>
```

### 3. 本機預覽（可選）

由於 OAuth redirect URI 必須與 GitLab OAuth App 設定完全相符，**本機測試建議使用 mock token**，或臨時在 OAuth App 加入 `http://localhost:8080/` 作為額外 redirect URI。

使用任意靜態伺服器：
```bash
# Python 3
python3 -m http.server 8080 --directory public

# 或使用 npx
npx serve public -p 8080
```

開啟瀏覽器至 `http://localhost:8080/publisher/`

---

## 部署至 GitLab Pages

Publisher 頁面跟隨整個 `nbu/public-pages` repo 的 CI/CD 一起部署：

```bash
# 確認目前在 main branch
git checkout main

# 加入 publisher 頁面
git add public/publisher/index.html
git commit -m "feat: add html page publisher"
git push origin main
```

Pipeline 跑完後，Publisher 頁面即可在以下網址存取：

```
https://public-pages-56dd79.fox.25sprout.com/publisher/
```

---

## 使用者操作流程

### 第一次使用

1. 開啟 `https://public-pages-56dd79.fox.25sprout.com/publisher/`
2. 點擊「**以 GitLab 帳號登入**」
3. 在 GitLab OAuth 頁面點擊「Authorize」
4. 自動跳回 Publisher，頂部顯示「已登入為 `{username}`」

### 發布 HTML 頁面

1. 點擊「**選擇 HTML 檔案**」，選取本機 `.html` 檔
2. 「**檔案名稱**」欄位會自動填入原始檔名（不含副檔名），可修改
3. 確認預覽網址：`https://nbu.fox.25sprout.com/public-pages/{username}/{filename}.html?v={hash}`（hash 在選擇檔案後即時顯示）
4. 點擊「**發布**」
5. 頁面顯示「部署中…」並開始輪詢 pipeline 狀態
6. Pipeline 成功後，顯示 CDN 生效倒數（10 秒），倒數結束後顯示「✓ 部署成功」及可點擊的網址

### 覆蓋已發布的頁面

使用 **相同檔案名稱** 重新上傳即可；系統偵測到同路徑檔案存在時自動執行覆蓋（不需手動確認）。

---

## 錯誤排查

| 錯誤訊息 | 原因 | 解決方式 |
|----------|------|----------|
| 登入失敗：安全驗證錯誤 | OAuth state 不符（可能為 CSRF 攻擊或舊 session） | 清除瀏覽器快取後重試 |
| 登入失敗：無法取得 token | GitLab OAuth App 設定錯誤、redirect URI 不符，或 App 被設為 Confidential | 請管理員確認 OAuth App 設定：Redirect URI 為 `https://public-pages-56dd79.fox.25sprout.com/publisher/`，且未勾選 Confidential |
| 登入已過期，請重新登入 | Access token 失效（通常 2 小時後） | 點擊登出後重新登入 |
| 僅支援 HTML 檔案 | 選取了非 .html 檔案 | 重新選取 .html 檔案 |
| 檔案名稱不合法 | 名稱含特殊字符（空白、斜線等） | 使用 `a-z 0-9 _ -` 字元命名 |
| 權限不足，請聯繫管理員 | 用戶未被加入專案或權限不夠 | 請 GitLab 管理員加入 Developer 權限 |
| 部署失敗 | GitLab CI pipeline 執行錯誤 | 點擊「查看 Pipeline 詳情」連結查看完整 log |
| 部署逾時 | Pipeline 超過 5 分鐘未完成 | 確認 GitLab Runner 是否正常運作；聯繫管理員 |

---

## 管理員備忘

- **新增用戶**：到 `nbu/public-pages` 專案成員設定加入 Developer 權限
- **查看發布歷史**：`https://fox.25sprout.com/nbu/public-pages/-/commits/main`（以 `publish:` 開頭的 commit 均為 Publisher 產生）
- **刪除用戶頁面**：直接在 GitLab 刪除 `public/{username}/` 目錄並 push 至 main
- **OAuth App 管理**：`https://fox.25sprout.com/admin/applications`
