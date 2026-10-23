# Tasks: HTML Page Publisher

**Input**: Design documents from `/specs/001-html-page-publisher/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/gitlab-api.md ✅, quickstart.md ✅

**Tech Stack**: Vanilla JavaScript (ES2020+), HTML5, CSS3；無 npm 套件、無建置步驟  
**Output**: `public/publisher/index.html`（單一自含式檔案）

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- 所有路徑均相對於 repo root

---

## Phase 1: Setup（專案初始化）

**Purpose**: 建立 Publisher 頁面的檔案結構與基礎 HTML 骨架

- [x] T001 建立 `public/publisher/index.html` 檔案，包含完整 HTML5 骨架（`<!doctype html>`、`<head>`、`<body>`）及 `<style>` / `<script>` 空白區塊
- [x] T002 在 `public/publisher/index.html` 的 `<script>` 頂部定義 `CONFIG` 常數物件（含 `GITLAB_BASE_URL`、`CLIENT_ID`、`REDIRECT_URI`、`PROJECT_PATH`、`PAGES_BASE_URL`、`PIPELINE_POLL_INTERVAL_MS`、`PIPELINE_TIMEOUT_MS`、`PUBLIC_DIR_PREFIX`），所有可設定值集中於此
- [x] T003 [P] 在 `public/publisher/index.html` 的 `<script>` 中定義 App 狀態機的狀態常數（`APP_STATE`：`UNAUTHENTICATED`、`OAUTH_CALLBACK`、`AUTHENTICATED`、`UPLOADING`、`POLLING`、`DEPLOYED`、`DEPLOYMENT_FAILED`、`DEPLOYMENT_TIMEOUT`）
- [x] T004 [P] 在 `public/publisher/index.html` 的 `<style>` 中加入基礎排版 CSS（page wrapper、nav、主要操作區塊、狀態顯示區塊、按鈕、錯誤訊息），風格參考既有的 `public/index.html`（深色調、匹配 NBU Pages 視覺風格）

**Checkpoint**: `public/publisher/index.html` 存在，瀏覽器可開啟但無任何功能

---

## Phase 2: Foundational（核心基礎設施）

**Purpose**: 實作所有 user story 共用的底層 UI 狀態管理與 GitLab API 封裝，MUST 在任何 user story 前完成

**⚠️ CRITICAL**: Phase 3、4、5 均依賴此 phase 完成

- [x] T005 在 `public/publisher/index.html` 中實作 `UI` 模組：`setState(newState)` 函式根據 App 狀態顯示/隱藏對應 UI 區塊（登入區、上傳表單區、pipeline 進度區）；`showError(message)` 顯示中文錯誤訊息；`showSuccess(message, url)` 顯示成功訊息與 URL；`setPublishBtnDisabled(bool)` 控制發布按鈕啟用狀態
- [x] T006 在 `public/publisher/index.html` 中實作 `GitLabAPI` 模組：`request(method, path, body)` 核心 Fetch 包裝函式，自動帶入 `Authorization: Bearer {token}` header；將 HTTP 401 映射為「登入已過期，請重新登入」、403 映射為「權限不足，請聯繫管理員開通發布權限」、其他 4xx/5xx 映射為「操作失敗：{status} {statusText}」；網路錯誤映射為「網路連線異常，請確認網路後重試」
- [x] T007 [P] 在 `public/publisher/index.html` 中實作 `FileOp.validate(file, targetFilename)` 函式：驗證 file.type 為 `text/html` 或副檔名為 `.html`/.htm`（FR-005）；驗證 targetFilename 僅含 `[a-zA-Z0-9_-]`、長度 1–100、不含 `/`、`\`、`..`、空白（路徑穿越防護）；檔案大小 <= 5MB；驗證失敗時回傳具體中文錯誤訊息

**Checkpoint**: 共用基礎設施就緒，UI 狀態切換可手動測試，API 模組可 console 測試

---

## Phase 3: User Story 2 — GitLab OAuth PKCE 登入（Priority: P2）

> **注意**：雖然 spec 標記 US2 為 P2，但登入是所有功能的前提。此 phase 必須在 US1 之前完成。

**Goal**: 用戶可透過 GitLab OAuth PKCE 流程登入，頁面顯示用戶名稱；可登出。

**Independent Test**: 開啟 `public/publisher/index.html`，點擊「以 GitLab 帳號登入」，完成 GitLab OAuth 授權後，頁面顯示「已登入為 {username}」及「登出」按鈕。點擊登出後回到未登入狀態。

### Implementation for User Story 2

- [x] T008 [US2] 在 `public/publisher/index.html` 中實作 `Auth.generatePKCE()` 非同步函式：以 `crypto.getRandomValues` 產生 64 bytes 作為 `code_verifier`（base64url 編碼）；以 `crypto.subtle.digest('SHA-256', ...)` 計算 `code_challenge`（base64url 編碼）；以 `crypto.getRandomValues` 產生 16 bytes 作為 `state`（hex 編碼）；回傳 `{ codeVerifier, codeChallenge, state }`
- [x] T009 [US2] 在 `public/publisher/index.html` 中實作 `Auth.initiateLogin()` 函式：呼叫 `generatePKCE()`；將 `codeVerifier` 存入 `sessionStorage['gl_pkce_verifier']`、`state` 存入 `sessionStorage['gl_oauth_state']`；以 `CONFIG` 組合 GitLab 授權 URL（含所有必要參數）並執行 `window.location.href` redirect（對應 contracts/gitlab-api.md §1a）
- [x] T010 [US2] 在 `public/publisher/index.html` 中實作 `Auth.handleCallback()` 非同步函式：解析 URL query string 中的 `code`、`state`、`error`；若無 code 則 return（非 callback 頁面）；驗證 `state` 與 `sessionStorage['gl_oauth_state']` 相符，否則顯示「登入失敗：安全驗證錯誤」並進入 ERROR 狀態；呼叫 GitLab token exchange API（contracts §1b）取得 access_token；清除 sessionStorage 中的 PKCE 中間狀態；儲存 `access_token` 至 `sessionStorage['gl_token']`；清理 URL query string（`history.replaceState`）
- [x] T011 [US2] 在 `public/publisher/index.html` 中實作 `Auth.fetchCurrentUser()` 非同步函式：呼叫 `GET /api/v4/user`（contracts §2）取得 `username`、`name`；將 `username` 儲存於 module-level 變數供後續使用；回傳 `{ username, name }`
- [x] T012 [US2] 在 `public/publisher/index.html` 中實作 `Auth.logout()` 函式：清除 `sessionStorage['gl_token']`、清除 module-level user 變數；呼叫 `UI.setState(APP_STATE.UNAUTHENTICATED)`
- [x] T013 [US2] 在 `public/publisher/index.html` 中加入 HTML 結構：未登入狀態的「以 GitLab 帳號登入」按鈕（`id="btn-login"`）；已登入狀態的用戶名稱顯示（`id="user-display"`）及「登出」按鈕（`id="btn-logout"`）；綁定點擊事件至 `Auth.initiateLogin()` 和 `Auth.logout()`
- [x] T014 [US2] 在 `public/publisher/index.html` 的頁面 `DOMContentLoaded` 事件中加入初始化邏輯：先呼叫 `Auth.handleCallback()`（處理 OAuth redirect 回來的情況）；若 `sessionStorage['gl_token']` 存在則嘗試 `Auth.fetchCurrentUser()` 恢復登入狀態；否則設定為 `UNAUTHENTICATED` 狀態

**Checkpoint**: US2 完成 — OAuth 登入/登出完整可運作，可獨立驗收

---

## Phase 4: User Story 1 — 上傳 HTML 並部署（Priority: P1）

**Goal**: 已登入用戶選擇 HTML 檔案、指定名稱、按下發布後，檔案被 commit 至 `public/{username}/{filename}.html`，pipeline 觸發。

**Independent Test**: 以已登入帳號選擇 `test.html`，填入名稱 `mytest`，點擊發布，確認 GitLab repo `public/{username}/mytest.html` 出現新 commit，且 pipeline 被觸發（可在 GitLab Web UI 驗證）。

### Implementation for User Story 1

- [x] T015 [US1] 在 `public/publisher/index.html` 中加入上傳表單 HTML 結構：檔案選擇 input（`id="file-input"`，`accept=".html,.htm"`）；檔案名稱 text input（`id="filename-input"`，placeholder 說明命名規則）；預覽網址 display（`id="url-preview"`，格式 `{PAGES_BASE_URL}/{username}/{filename}.html`）；「發布」按鈕（`id="btn-publish"`）；inline 驗證錯誤區塊（`id="validation-error"`）；整個表單區塊初始為 hidden，登入後顯示
- [x] T016 [US1] 在 `public/publisher/index.html` 中綁定 `#file-input` `change` 事件：以 filename（不含副檔名）自動填入 `#filename-input`（若欄位為空）；即時更新 `#url-preview`顯示的預覽網址
- [x] T017 [US1] 在 `public/publisher/index.html` 中綁定 `#filename-input` `input` 事件：即時更新 `#url-preview`；若輸入值含非法字符則即時顯示 inline 提示（但不阻擋輸入）
- [x] T018 [US1] 在 `public/publisher/index.html` 中實作 `FileOp.readAsBase64(file)` 非同步函式：以 `FileReader.readAsDataURL` 讀取檔案，擷取 base64 資料部分（去除 `data:text/html;base64,` 前綴）；回傳 base64 字串
- [x] T019 [US1] 在 `public/publisher/index.html` 中實作 `GitLabAPI.checkFileExists(filePath)` 非同步函式：呼叫 `GET /api/v4/projects/{PROJECT_PATH}/repository/files/{encodedPath}?ref=main`（contracts §3）；return `{ exists: true, lastCommitId: '...' }` 或 `{ exists: false }`；非 404 錯誤向上拋出
- [x] T020 [US1] 在 `public/publisher/index.html` 中實作 `GitLabAPI.commitFile(filePath, base64Content, operation, lastCommitId, username)` 非同步函式：`operation === 'create'` 時呼叫 POST（contracts §4a）；`operation === 'update'` 時呼叫 PUT（contracts §4b，帶入 `last_commit_id`）；`commit_message` 格式：`publish: {operation} {username}/{filename}.html`；回傳成功回應
- [x] T021 [US1] 在 `public/publisher/index.html` 中實作「發布」按鈕的 `handlePublish()` 非同步函式（主流程）：1) 呼叫 `FileOp.validate()` 驗證，失敗則顯示錯誤並 return；2) `UI.setPublishBtnDisabled(true)`，切換至 `UPLOADING` 狀態；3) `FileOp.readAsBase64(file)`；4) `checkFileExists(targetPath)` 決定 operation；5) `commitFile(...)` 執行 commit；6) 成功後切換至 `POLLING` 狀態並呼叫 `PipelinePoller.start()`；7) 捕捉所有錯誤，401 → 清除 token 並導向登入，403 → 顯示權限不足，其他 → 顯示具體錯誤訊息，並 `UI.setPublishBtnDisabled(false)`
- [x] T022 [US1] 在 `public/publisher/index.html` 中加入 `UPLOADING` 狀態的 UI 元素：pipeline 進度區塊（`id="pipeline-status"`，初始 hidden）；狀態文字（`id="pipeline-status-text"`）；GitLab pipeline 連結（`id="pipeline-link"`）；成功後的部署網址顯示（`id="deployed-url"`）

**Checkpoint**: US1 完成 — 上傳發布流程端對端可運作，可獨立驗收（pipeline 觸發確認）

---

## Phase 5: User Story 3 — 即時 Pipeline 進度（Priority: P3）

**Goal**: 部署觸發後，頁面自動輪詢 pipeline 狀態（每 8 秒），顯示當前進度；成功顯示 URL，失敗顯示錯誤訊息；5 分鐘逾時停止輪詢。

**Independent Test**: 觸發一次部署，確認畫面出現「等待中…」→「部署中…」→「部署成功 ✓」的狀態轉換，每次轉換均自動發生（無需手動刷新）；最終顯示可點擊的部署網址。

### Implementation for User Story 3

- [x] T023 [US3] 在 `public/publisher/index.html` 中實作 `GitLabAPI.getLatestPipeline()` 非同步函式：呼叫 `GET /api/v4/projects/{PROJECT_PATH}/pipelines?ref=main&per_page=1&order_by=id&sort=desc`（contracts §5）；回傳 pipeline 物件或 null（陣列為空時）
- [x] T024 [US3] 在 `public/publisher/index.html` 中實作 `GitLabAPI.getPipelineStatus(pipelineId)` 非同步函式：呼叫 `GET /api/v4/projects/{PROJECT_PATH}/pipelines/{id}`（contracts §6）；回傳 `{ id, status, webUrl, finishedAt }`
- [x] T025 [US3] 在 `public/publisher/index.html` 中實作 `PipelinePoller` 模組：`start(deployedFilePath)` 函式啟動輪詢 — 先等待 1.5 秒再呼叫 `getLatestPipeline()` 取得 pipeline ID（若空則重試最多 3 次）；將 pipeline ID 存入 `sessionStorage['gl_pipeline_id']` 供頁面重整後恢復；啟動 `setInterval`（`CONFIG.PIPELINE_POLL_INTERVAL_MS`）呼叫 `poll()`；同時啟動 `setTimeout`（`CONFIG.PIPELINE_TIMEOUT_MS`）呼叫 `onTimeout()`
- [x] T026 [US3] 在 `public/publisher/index.html` 中實作 `PipelinePoller.poll()` 函式：呼叫 `getPipelineStatus(id)`；更新 `#pipeline-status-text` 為對應的中文狀態文字（依 data-model.md 顯示映射表）；更新 `#pipeline-link` href 為 `webUrl`；若 status 為終止狀態（`success`/`failed`/`canceled`/`skipped`）呼叫 `stop()` 並觸發對應的 `onSuccess()`/`onFailed()`/`onCanceled()`；網路錯誤時僅顯示 retry 警告，不停止輪詢
- [x] T027 [US3] 在 `public/publisher/index.html` 中實作 `PipelinePoller.onSuccess(deployedFilePath)` 函式：`stop()` 停止輪詢；計算完整部署 URL（`CONFIG.PAGES_BASE_URL/{username}/{filename}.html`）；切換至 `DEPLOYED` 狀態；`UI.showSuccess('部署成功 ✓', deployedUrl)`；更新 `#deployed-url` 為可點擊連結；`UI.setPublishBtnDisabled(false)` 並顯示「再次發布」UI
- [x] T028 [US3] 在 `public/publisher/index.html` 中實作 `PipelinePoller.onFailed()` 函式：`stop()`；切換至 `DEPLOYMENT_FAILED` 狀態；`UI.showError('部署失敗，請點擊下方連結查看 Pipeline 錯誤詳情')`；確保 `#pipeline-link` 可見；`UI.setPublishBtnDisabled(false)`
- [x] T029 [US3] 在 `public/publisher/index.html` 中實作 `PipelinePoller.onTimeout()` 函式：`stop()`；切換至 `DEPLOYMENT_TIMEOUT` 狀態；`UI.showError('部署逾時（超過 5 分鐘），請至 GitLab 確認 Pipeline 狀態或聯繫管理員')`；`UI.setPublishBtnDisabled(false)`
- [x] T030 [US3] 在 `public/publisher/index.html` 的頁面初始化邏輯中加入 pipeline 恢復：若 `sessionStorage['gl_pipeline_id']` 存在且用戶已登入，自動呼叫 `PipelinePoller.resumeFromSession()`（使用已存的 pipeline ID 繼續輪詢，而非重新查詢最新 pipeline）

**Checkpoint**: US3 完成 — Pipeline 進度追蹤完整可運作，含 5 分鐘逾時與錯誤顯示

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: 安全強化、UX 細節、完整性驗收

- [x] T031 [P] 在 `public/publisher/index.html` 中確認路徑注入防護完整實作：在 `handlePublish()` 中再次驗證 `targetPath` 的 `username` 區段確實等於從 token 取得的 `current_user.username`（對應 FR-006、data-model.md 驗證規則）；若不符（理論上不可能，作為 defence-in-depth）顯示錯誤並中止
- [x] T032 [P] 在 `public/publisher/index.html` 中加入 `filePath` URL encode 工具函式 `encodeFilePath(path)`：將路徑中的 `/` 替換為 `%2F`，並對每個 segment 執行 `encodeURIComponent`（確保特殊字符被正確編碼）；所有呼叫 GitLab API 的路徑均使用此函式
- [x] T033 對 `public/publisher/index.html` 進行完整 UX review 並修正：確認所有中文文案一致（「發布」而非「上傳」、「部署」而非「佈署」）；確認所有錯誤情境（Edge Cases in spec.md）均有對應的 UI 回饋；確認 `#url-preview` 在未選擇檔案或未輸入名稱時顯示提示而非空白
- [x] T034 更新 `public/index.html` 首頁，在適當位置加入「發布你的頁面」連結（指向 `/public-pages/publisher/`），讓 Publisher 工具可被發現

---

## Phase 7: Post-delivery UX Improvements（交付後持續改善）

**Purpose**: 部署上線後根據實際使用情境追加的 UX 改善項目

- [x] T035 [P] 在 `public/publisher/index.html` 中加入 pure-JS SHA-256 fallback 函式 `_sha256Sync(input: Uint8Array): Uint8Array`：以手寫 SHA-256 演算法實作，不依賴 `crypto.subtle`；`_sha256(bytes)` wrapper 優先使用 `crypto.subtle.digest`（HTTPS 環境），fallback 至 `_sha256Sync`（HTTP + 自訂 hostname 本機開發環境）
- [x] T036 [P] 將 `CONFIG.REDIRECT_URI` 從本機開發值更新為正式生產值 `https://public-pages-56dd79.fox.25sprout.com/publisher/`（GitLab Unique Domain，繞過 path 限制）；同步更新 GitLab OAuth Application 的 Redirect URI 設定
- [x] T037 在 `public/publisher/index.html` 中於 `PipelinePoller.onSuccess()` 加入 CDN 生效倒數計時 UI：pipeline 成功後先隱藏部署網址與「再次發布」按鈕，顯示 `CONFIG.CDN_WARMUP_MS`（預設 10 秒）倒數計時，倒數結束後再顯示網址；`CONFIG` 新增 `CDN_WARMUP_MS: 10000` 常數
- [x] T038 在 `public/publisher/index.html` 的 `handlePublish()` 及 `PipelinePoller.onSuccess()` 中加入 cache-busting `?v={hash}` 邏輯：計算 HTML 檔案 base64 內容的 SHA-256，取前 2 bytes 轉為 4 字元十六進位字串（如 `a3f2`）；`PipelinePoller.start(deployedFilePath, fileHash)` 新增 `fileHash` 參數；`sessionStorage['gl_file_hash']` 持久化供 pipeline 恢復後使用；`onSuccess()` 建構的 `deployedUrl` 加上 `?v={hash}`
- [x] T039 在 `public/publisher/index.html` 中將 hash 計算時機前移至 `fileInput.change` async 事件 handler：選取檔案後即時計算 `selectedFileHash`，存入 module-level 變數；`UI.updateUrlPreview()` 若 `selectedFileHash` 非 null 則在預覽網址後加上 `?v={hash}`；`handlePublish()` 改為直接重用 `selectedFileHash` 而非重新計算；logout 與「再次發布」時清除 `selectedFileHash`

---

## Dependencies

```
T001 → T002 → T003, T004（可並行）
T005, T006, T007（可並行，均依賴 T001-T002）
T008 → T009 → T010 → T011 → T012 → T013 → T014（US2 線性流程）
T014（US2 完成）→ T015 → T016, T017（可並行）→ T018, T019（可並行）→ T020 → T021 → T022
T022（US1 完成）→ T023 → T024 → T025 → T026 → T027, T028, T029（可並行）→ T030
T031, T032（可並行，在 T021 後）→ T033 → T034
```

**Parallel Opportunities per Story**:
- **Phase 1**: T003 // T004（UI 骨架與 CSS 可並行）
- **Phase 2**: T005 // T006 // T007（三個模組互相獨立）
- **Phase 4 US1**: T016 // T017（兩個事件綁定獨立）；T018 // T019（FileReader 與 API check 獨立）
- **Phase 5 US3**: T023 // T024（兩個 API 函式獨立）；T027 // T028 // T029（三個 callback 獨立）
- **Phase 6**: T031 // T032（安全強化兩個面向獨立）

---

## Implementation Strategy

**MVP Scope（最小可展示）**：Phase 1 + Phase 2 + Phase 3（US2 登入）+ Phase 4（US1 上傳）= T001–T022

- 完成後用戶可登入並成功發布 HTML 頁面，pipeline 被觸發（可手動到 GitLab 查看狀態）
- 不含即時進度追蹤（Phase 5）

**Full Scope**：T001–T034（含 pipeline 即時進度 + polish）

**建議實作順序**:
1. Phase 1 → Phase 2（基礎設施）
2. Phase 3（US2 登入）— 有 OAuth 才能測試後續所有功能
3. Phase 4（US1 上傳）— 核心 MVP 功能
4. Phase 5（US3 進度追蹤）— 體驗提升
5. Phase 6（polish）— 最後補全

---

## 總計

| Phase | Tasks | User Story |
|-------|-------|------------|
| Phase 1: Setup | T001–T004 | 4 tasks |
| Phase 2: Foundational | T005–T007 | 3 tasks |
| Phase 3: US2 OAuth 登入 | T008–T014 | 7 tasks |
| Phase 4: US1 上傳發布 | T015–T022 | 8 tasks |
| Phase 5: US3 Pipeline 進度 | T023–T030 | 8 tasks |
| Phase 6: Polish | T031–T034 | 4 tasks |
| Phase 7: Post-delivery UX | T035–T039 | 5 tasks |
| **Total** | **T001–T039** | **39 tasks** |

**Parallel opportunities**: 14 tasks 標記 [P]  
**MVP cutoff**: T001–T022（22 tasks）
