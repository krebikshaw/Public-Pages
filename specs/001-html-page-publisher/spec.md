# Feature Specification: HTML Page Publisher

**Feature Branch**: `001-html-page-publisher`  
**Created**: 2026-04-02  
**Status**: Complete  
**Input**: User description: "我要做一個前端畫面，目標是跑在 gitlab page 上（自架的 gitlab）。目的是，讓他上傳一個 html 檔可以自動 commit 進 repo 自動跑 pipeline 佈署成 gitlab page，然後給一個網址。功能需要 gitlab oauth 取得用戶 token，然後用戶名就是網址的 path 之一。然後可以指定檔案名，若檔案名存在則覆蓋。然後要在畫面上可以隨時看到該 pipeline 跑的進度，若有錯誤則噴錯讓用戶知道。"

## Clarifications

### Session 2026-04-02

- Q: 路徑中的 `{username}` 是由系統從 OAuth token 強制綁定，還是由用戶自行輸入？ → A: 系統從 OAuth token 取得目前登入的 username 並強制組合為路徑第一段，UI 不顯示也不允許修改 username 欄位。
- Q: OAuth 流程是純前端 PKCE 還是需要後端 server 處理 token 交換？ → A: 使用 OAuth 2.0 PKCE 流程，純靜態頁面完成，無需後端 server 或 client secret。
- Q: Pipeline 輪詢的最長等待上限（逾時時間）為多久？ → A: 5 分鐘，逾時後停止輪詢並顯示「部署逾時」。
- Q: Commit 操作使用用戶自己的 token 還是 service account token？ → A: 用戶自己的 token，用戶須具備對共用專案的 Developer 權限。
- Q: 用戶能否在頁面上查看自己過去發布過的頁面清單？ → B: Out of scope，本次不開發清單功能；用戶若需查看歷史發布記錄，請自行進入 GitLab repo 查看。

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 上傳 HTML 並部署為 GitLab Page (Priority: P1)

已登入的用戶選取本機的 HTML 檔案，指定一個檔案名稱，按下發布後，系統自動將檔案 commit 至後端 GitLab 專案並觸發 CI/CD pipeline；部署成功後，畫面顯示可存取的 GitLab Pages URL。

**Why this priority**: 這是整個功能的核心價值所在。若只實作此段，用戶已能完整體驗「上傳 → 部署 → 取得網址」的完整流程，可立即作為 MVP 演示。

**Independent Test**: 以已登入的測試帳號選擇一個 `index.html` 並按下發布，確認 pipeline 觸發、完成後畫面出現能訪問該頁面的網址，即為成功。

**Acceptance Scenarios**:

1. **Given** 用戶已登入且尚無任何已發布頁面，**When** 選擇本機 HTML 檔、輸入檔案名稱後點擊「發布」，**Then** 系統顯示「佈署中」狀態並呈現 pipeline 進度，最終顯示部署成功及可存取的 GitLab Pages URL。
2. **Given** 用戶已登入且目的地已有相同檔案名稱的頁面，**When** 使用相同檔案名稱再次上傳新版本，**Then** 舊版本被覆蓋，pipeline 觸發成功，URL 不變但內容更新。
3. **Given** 上傳的 HTML 檔案內含語法錯誤導致 pipeline 失敗，**When** pipeline 執行失敗，**Then** 畫面顯示明確的錯誤訊息及失敗原因，用戶不會收到錯誤的成功 URL。

---

### User Story 2 - 以 GitLab OAuth 登入 (Priority: P2)

未登入的訪客進入頁面，點擊「以 GitLab 帳號登入」，系統導向自架 GitLab 的 OAuth 授權頁面；用戶授權後，系統取得 access token 並顯示用戶名稱，進入已登入狀態。

**Why this priority**: 登入是所有操作的前提，但因為可以用 mock token 繞過進行核心功能開發，所以優先序略低於核心部署流程。

**Independent Test**: 可透過點擊「以 GitLab 帳號登入」完成 OAuth 流程，確認頁面顯示「登入為 {username}」及登出按鈕，視為獨立可測試。

**Acceptance Scenarios**:

1. **Given** 未登入的訪客進入頁面，**When** 點擊「以 GitLab 帳號登入」，**Then** 頁面導向 GitLab OAuth 授權頁面。
2. **Given** 用戶在 GitLab 完成授權，**When** 被重定向回本頁面，**Then** 頁面顯示「已登入為 {gitlab-username}」且發布功能可用。
3. **Given** 用戶已登入，**When** 點擊「登出」，**Then** token 被清除，頁面回到未登入狀態，發布功能不可用。
4. **Given** OAuth 授權被用戶拒絕，**When** 被重定向回本頁面，**Then** 頁面顯示登入失敗提示並維持未登入狀態。

---

### User Story 3 - 即時查看 Pipeline 執行進度 (Priority: P3)

在 HTML 檔案上傳並觸發 pipeline 後，用戶無需離開頁面，可在同一畫面看到 pipeline 目前的狀態（等待中、執行中、成功、失敗），每個 stage 的進度，以及完整的錯誤訊息（若失敗）。

**Why this priority**: 進度追蹤提升用戶體驗，但用戶仍可手動去 GitLab 查看 pipeline，因此非 MVP 必備，但優先開發。

**Independent Test**: 觸發一次部署後，確認畫面出現 pipeline 進度指示器並每隔幾秒自動更新狀態，最終顯示成功或失敗結果，即為成功。

**Acceptance Scenarios**:

1. **Given** 部署已觸發，**When** pipeline 正在執行，**Then** 畫面顯示目前 stage 名稱及整體狀態（running），並定期自動更新，無需手動刷新。
2. **Given** pipeline 執行成功，**When** 狀態更新為成功，**Then** 畫面顯示「部署成功」、GitLab Pages URL，並停止輪詢。
3. **Given** pipeline 執行失敗，**When** 狀態更新為失敗，**Then** 畫面顯示「部署失敗」及從 pipeline 取得的錯誤訊息，方便用戶診斷問題。
4. **Given** 網路中斷，**When** 輪詢 pipeline 狀態失敗，**Then** 畫面顯示「無法取得 pipeline 狀態，請重新整理」，不產生例外崩潰。

---

### Edge Cases

- 用戶上傳的檔案不是 `.html` 副檔名時，系統應拒絕並提示「僅支援 HTML 檔案」。
- 用戶輸入的檔案名稱包含特殊字符（空白、斜線、`..` 等路徑穿越字符）時，系統應拒絕並提示合法命名規則。
- 用戶 OAuth token 已過期，在點擊發布時，系統應提示「登入已過期，請重新登入」並清除 token。
- pipeline 觸發後，用戶關閉頁面再重新開啟，系統應允許用戶重新查詢最後一次 pipeline 狀態（依 session 或 URL 參數恢復）。
- 同一用戶同時觸發多次部署（例如連點發布按鈕），系統應防止重複提交，發布按鈕在處理中應為不可點擊狀態。
- 若後端 GitLab 專案的 pipeline 環境設定有誤（例如缺少 runner），pipeline 永遠不啟動，系統應在**輪詢 5 分鐘後**顯示「部署逾時，請聯繫管理員」並停止輪詢。
- 系統不接受任何來自用戶端對 commit 路徑 username 段的修改；若請求中偵測到路徑前綴與 token 所屬 username 不符，系統應立即拒絕操作。
- 用戶的 token 對共用專案權限不足（未被加入為 Developer）時，系統應顯示「權限不足，請聯繫管理員開通發布權限」，不暴露原始 403 錯誤碼。

## Requirements *(mandatory)*

### Functional Requirements

**Out of Scope**:
- 已發布頁面的游歷清單 / 歷史記錄（用戶若需略覽歷史，請自行到 GitLab repo 查看 commit 記錄）。
- 刪除已發布的頁面。
- 多檔同時上傳發布。

- **FR-001**: 系統 MUST 提供 GitLab OAuth 2.0 **PKCE 流程**登入，讓用戶以自架 GitLab 帳號完成身份驗證以取得 access token；整個流程須在純靜態前端頁面內完成，不依賴任何後端 server 處理 token 交換。
- **FR-002**: 系統 MUST 在頁面上顯示目前登入的 GitLab 用戶名稱，並提供登出功能。
- **FR-003**: 用戶 MUST 能夠從本機選取一個 HTML 檔案進行上傳。
- **FR-004**: 用戶 MUST 能夠指定發布後的檔案名稱（不含副檔名）；系統應預設以原始檔案名稱填入。
- **FR-005**: 系統 MUST 驗證上傳的檔案為 HTML 格式，並驗證指定的檔案名稱符合安全命名規則（不含路徑穿越字符）。
- **FR-006**: 系統 MUST 使用用戶自己的 GitLab access token，透過 `https://fox.25sprout.com` 的 GitLab API 將 HTML 檔案 commit 至共用專案 `nbu/public-pages`（`git@fox.25sprout.com:nbu/public-pages.git`），存放路徑為 `{username}/{filename}.html`。路徑中的 `username` 由系統從 OAuth token 自動取得並強制帶入，UI 不提供 username 欄位，用戶無法修改或偽造路徑前綴。用戶需具備該共用專案的 Developer 或以上權限，系統應於權限不足時顯示明確提示。
- **FR-007**: 若目的地路徑已有相同檔案名稱，系統 MUST 執行覆蓋（更新）操作，而非建立重複檔案。
- **FR-008**: Commit 完成後，系統 MUST 自動觸發目標 GitLab 專案的 CI/CD pipeline。
- **FR-009**: 系統 MUST 在頁面上即時顯示 pipeline 的執行狀態（pending、running、success、failed），並定期自動輪詢更新，無需用戶手動刷新。
- **FR-010**: 若 pipeline 執行失敗，系統 MUST 顯示來自 pipeline 的錯誤訊息，幫助用戶了解失敗原因。
- **FR-011**: Pipeline 成功後，系統 MUST 顯示已部署頁面的 GitLab Pages URL（URL 路徑中包含用戶名稱），並附加 `?v={hash}` 查詢字串用於快取破壞（cache-busting）。`hash` 為檔案 base64 內容的 SHA-256 前 2 bytes（4 個十六進位字元），在選擇檔案時即時計算並顯示於預覽網址。
- **FR-012**: 發布操作進行中，發布按鈕 MUST 為禁用狀態，防止重複提交。
- **FR-013**: Pipeline 成功後，系統 MUST 顯示 CDN 生效倒數計時（預設 10 秒），在此期間隱藏部署網址與「再次發布」按鈕，倒數結束後再顯示，以避免 CDN 尚未更新時點開連結出現 404。
- **FR-014**: 系統 MUST 在非 HTTPS 環境（HTTP + 自訂 hostname 的本機開發）中提供純 JS SHA-256 fallback，確保 `crypto.subtle` 不可用時 `?v={hash}` 功能仍可運作。

### Key Entities

- **User（用戶）**: 代表一位通過 GitLab OAuth 認證的使用者；包含 GitLab 用戶名稱（username）、access token；token 僅存於用戶端 session，不持久化至伺服器。
- **HTMLPage（發布頁面）**: 代表用戶發布的一個 HTML 頁面；包含檔案名稱、目標路徑（`{username}/{filename}.html`）、發布時間；對應 GitLab repo 中的一個檔案。
- **Pipeline（部署任務）**: 代表一次 GitLab CI/CD pipeline 執行；包含 pipeline ID、狀態（pending/running/success/failed）、所屬 stage 資訊、錯誤訊息；與 HTMLPage 的發布動作一對一關聯。
- **DeploymentURL（部署網址）**: Pipeline 成功後由 GitLab Pages 提供的公開 URL，格式為 `https://nbu.fox.25sprout.com/public-pages/{username}/{filename}.html?v={hash}`，其中 `hash` 為 4 字元十六進位字串（SHA-256 前 2 bytes），若無法計算則省略 `?v=` 部分。

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 用戶從選好檔案到取得 GitLab Pages URL（pipeline 成功後）的全程操作，在正常網路環境下可在 5 分鐘內完成。
- **SC-002**: 發布操作的成功率達 95% 以上（排除 GitLab 服務本身故障的情況）。
- **SC-003**: Pipeline 狀態頁面每 10 秒以內自動更新一次，用戶無需手動刷新即可知道最新進度。
- **SC-004**: 95% 的初次使用者可以在無說明文件的情況下，自主完成登入 → 上傳 → 取得 URL 的完整流程。
- **SC-005**: 所有可預期的錯誤情境（檔案格式錯誤、命名不合法、token 過期、pipeline 失敗）均以用戶可理解的中文提示呈現，用戶不會看到原始的技術性錯誤碼。
- **SC-006**: Pipeline 輪詢至多埵續 5 分鐘，逾時後系統必須自動停止輪詢並顯示明確的「部署逾時」提示，不得無限等待。

## Assumptions

- 本功能本身以靜態 HTML 頁面形式部署於自架 GitLab Pages，由此頁面作為操作入口（即工具本身也是 GitLab Pages 上的一個頁面）。
- 目標 GitLab 專案為 `nbu/public-pages`（`git@fox.25sprout.com:nbu/public-pages.git`），GitLab 實例為 `https://fox.25sprout.com`；其 `.gitlab-ci.yml` 已配置好 GitLab Pages 部署邏輯，用戶上傳的檔案被 commit 後即可觸發 pipeline。
- GitLab 管理員已在 `https://fox.25sprout.com` 上建立 OAuth Application，提供 `client_id` 與授權端點，並啟用 PKCE 支援（GitLab 13.0+），供本功能進行無後端的 OAuth PKCE 流程；**不需要 client secret**。
- GitLab 管理員已將所有將使用本功能的 GitLab 用戶加入共用專案並授予 Developer 或以上權限，不具備權限的用戶無法發布。
- 用戶名稱已被規範化為 GitLab 的合法 username，可直接用作 URL path 的一部分，無需額外轉換。
- 部署的 GitLab Pages base URL 為 `https://nbu.fox.25sprout.com/public-pages/`，完整 URL 格式為 `https://nbu.fox.25sprout.com/public-pages/{username}/{filename}.html`，可在 pipeline 成功後直接組合，無需額外 API 查詢。
- 本工具不儲存或代理 GitLab access token，token 僅存於用戶瀏覽器的 session storage，隨瀏覽器標籤關閉而消失。
