# Feature Specification: 上傳頁已發布檔案管理清單

**Feature Branch**: `002-upload-file-manager`  
**Created**: 2026-04-16  
**Status**: Implemented  
**Input**: User description: "我想在上傳頁放一區用戶上傳過的檔案清單，可以直接點擊開新視窗、取得連結、刪除等功能"

## Assumptions

- 已登入的用戶才能看到此清單區塊（與整個 publisher 頁面的登入前提一致）。
- 「已上傳的檔案」定義為：目前在 GitLab repo `public/{username}/` 目錄下的所有 `.html` 檔案（透過 GitLab API 查詢）。
- 「取得連結」的 URL 格式與 spec 001 部署成功後回傳的 URL 格式相同（`https://{gitlab-pages-domain}/{username}/{filename}`）。
- 刪除操作會透過 GitLab API 從 repo 中移除對應檔案，並需觸發一次新的 pipeline 重新部署，確保對應 URL 失效。
- 清單在頁面載入及每次上傳完成後自動刷新。

## User Scenarios & Testing *(mandatory)*

### User Story 1 - 查看已發布的頁面清單 (Priority: P1)

已登入的用戶在上傳頁看到「我的已發布頁面」區塊，列出自己所有已發布的 HTML 檔案，每筆顯示檔名與對應的 GitLab Pages URL。

**Why this priority**: 這是整個功能的基礎。沒有清單，後續的開啟、複製連結、刪除動作都無從操作。且本故事本身就提供「確認過去是否已發布某頁面」的即時價值。

**Independent Test**: 登入後在上傳頁確認出現清單區塊，且列出的檔案與 GitLab repo `public/{username}/` 目錄的實際內容一致，即為成功。

**Acceptance Scenarios**:

1. **Given** 用戶已登入且 `public/{username}/` 目錄下有 2 個 HTML 檔，**When** 頁面載入完成，**Then** 清單區塊顯示這 2 個檔名，依最後更新時間由新至舊排列，每列提供「開啟」「複製連結」「刪除」三個操作按鈕。
2. **Given** 用戶已登入但尚未發布任何頁面，**When** 頁面載入完成，**Then** 清單區塊顯示「尚無已發布的頁面」提示，而非空白或錯誤。
3. **Given** 用戶成功上傳一個新檔案，**When** 部署完成後，**Then** 清單自動刷新並顯示新增的項目。
4. **Given** 用戶未登入，**When** 進入頁面，**Then** 清單區塊不顯示（與整個 publisher 區塊一致）。

---

### User Story 2 - 點擊開新視窗預覽頁面 (Priority: P2)

用戶在清單中對某筆項目點擊「開啟」，瀏覽器以新分頁（新視窗）開啟對應的 GitLab Pages URL。

**Why this priority**: 快速預覽是最常見的使用情境，優先序僅次於清單本身。

**Independent Test**: 在清單中對任一項目點擊「開啟」按鈕，確認瀏覽器新分頁成功開啟對應的 GitLab Pages URL，即為成功。

**Acceptance Scenarios**:

1. **Given** 清單中顯示一個已發布的 `demo.html`，**When** 用戶點擊該列的「開啟」按鈕，**Then** 瀏覽器以新分頁開啟 `https://{pages-domain}/{username}/demo.html`。
2. **Given** GitLab Pages 服務暫時不可用，**When** 用戶在新分頁開啟 URL，**Then** 瀏覽器顯示服務不可用訊息（此為外部服務行為，不在本系統控制範圍內）。

---

### User Story 3 - 複製頁面連結 (Priority: P2)

用戶在清單中對某筆項目點擊「複製連結」，系統將對應的 GitLab Pages URL 複製到剪貼簿，並顯示短暫的「已複製！」提示回饋。

**Why this priority**: 複製連結是分享的核心動作，與「開新視窗」同等重要，並列 P2。

**Independent Test**: 點擊「複製連結」後貼到任意文字輸入框，確認貼上的 URL 與清單顯示的 URL 完全一致，即為成功。

**Acceptance Scenarios**:

1. **Given** 清單顯示 `report.html`，**When** 點擊「複製連結」，**Then** 剪貼簿內容為該頁面 URL 並附加隨機 cache-busting 參數（格式：`https://…/alice/report.html?v=a1b2`），且按鈕短暫顯示「已複製！」後恢復原文字。每次點擊產生的 `?v=` 值皆不同，確保每次複製的連結都能繞過瀏覽器快取。
2. **Given** 瀏覽器不支援 Clipboard API（安全策略限制），**When** 點擊「複製連結」，**Then** 系統以 `execCommand` fallback 嘗試寫入剪貼簿；若仍失敗，則以 toast 顯示含 `?v=` 的完整 URL 供用戶手動複製，不應靜默失敗。

---

### User Story 4 - 刪除已發布的頁面 (Priority: P3)

用戶在清單中對某筆項目點擊「刪除」，系統顯示確認對話框；確認後，系統透過 GitLab API 刪除對應檔案，清單即時移除該項目。

**Why this priority**: 刪除是破壞性操作，需要額外的確認流程，開發複雜度較高，且並非每次使用都需要。P1/P2 完成後再補充。

**Independent Test**: 點擊某項目的「刪除」並確認後，重新查詢 GitLab repo 確認該檔案已不存在，且清單中也不再顯示該項目，即為成功。

**Acceptance Scenarios**:

1. **Given** 清單中存在 `old-page.html`，**When** 用戶點擊「刪除」，**Then** 顯示「確定要刪除 old-page.html 嗎？此操作無法復原。」的確認對話框，提供「確認刪除」與「取消」兩個按鈕。
2. **Given** 確認對話框已顯示，**When** 用戶點擊「確認刪除」，**Then** 系統呼叫 GitLab API 刪除檔案、清單移除該列、顯示「已成功刪除 old-page.html」提示。
3. **Given** 確認對話框已顯示，**When** 用戶點擊「取消」，**Then** 對話框關閉，清單無任何變化。
4. **Given** 刪除時 API 回傳錯誤（如 403 權限不足），**When** 刪除操作失敗，**Then** 顯示「刪除失敗，請稍後再試」錯誤提示，清單維持原狀。
5. **Given** 用戶 token 已過期，**When** 嘗試刪除，**Then** 系統提示「登入已過期，請重新登入」並清除 token，同 spec 001 的 token 過期處理。

---

### Edge Cases

- 兩個瀏覽器分頁同時開啟 publisher 頁，用戶在分頁 A 刪除某檔案後，分頁 B 的清單不會自動更新（不在本功能範圍），但分頁 B 再次手動刷新後應反映最新狀態。
- 若 GitLab API 查詢清單時返回錯誤（如網路逾時），應顯示「無法載入清單，請重新整理頁面」，不應崩潰或顯示空白。
- 清單中檔名顯示超長時（如 50 字元以上），應進行截斷或 ellipsis 處理，不破壞排版。
- 清單項目超過 20 筆時，顯示前 20 筆並提供分頁或捲動區域供用戶瀏覽其餘項目。

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: 系統 MUST 在上傳頁的登入狀態下顯示「已發布頁面清單」區塊。
- **FR-002**: 系統 MUST 透過 GitLab API（Repository Trees API）列出 `public/{username}/` 目錄下的所有 `.html` 檔案。
- **FR-003**: 清單每列 MUST 顯示：檔名（加粗，超長時 ellipsis 截斷）、「開啟」按鈕、「複製連結」按鈕、「刪除」按鈕。列內不顯示完整 URL，以保持排版簡潔。
- **FR-004**: 點擊「開啟」MUST 以 `target="_blank"` 在新分頁開啟對應 URL。
- **FR-005**: 點擊「複製連結」MUST 將 URL（附加隨機 `?v=xxxx` cache-busting 參數，由 `crypto.getRandomValues` 產生 4 位 hex）複製至剪貼簿，並顯示不少於 1 秒的成功回饋提示。每次點擊產生不同的 `?v=` 值。
- **FR-006**: 點擊「刪除」MUST 先顯示確認對話框，用戶確認後才執行刪除 API 呼叫。
- **FR-007**: 刪除成功後，系統 MUST 立即從清單移除對應項目，無需整頁刷新。
- **FR-008**: 刪除檔案後，系統 MUST 自動觸發一次新的 pipeline 重新部署，確保對應的 GitLab Pages URL 失效。
- **FR-009**: 上傳新檔案部署完成後，系統 MUST 自動刷新清單。
- **FR-010**: 清單最多顯示 20 筆（取時間排序後的前 20 筆）。清單容器固定高度（`max-height: 400px`），超出時以垂直捲動瀏覽，不使用分頁。
- **FR-013**: 清單 MUST 依檔案最後更新時間由新至舊排列。排序依據為 GitLab Commits API（`GET /repository/commits?path=public/{username}`）回傳的 commit 歷史，解析 commit message（格式：`publish: create/update {username}/{filename}.html`）提取各檔案最近一次 commit 順序；若 Commits API 不可用則回退為 Trees API 的原始順序。
- **FR-011**: 清單載入中應顯示載入指示（loading state），避免空白閃爍。
- **FR-012**: 用戶未登入時，清單區塊 MUST NOT 顯示。

### Key Entities

- **PublishedFile**: 代表一個已發布的 HTML 頁面，屬性包含 `filename`（檔名）、`pageUrl`（GitLab Pages 完整 URL）、`filePath`（repo 內相對路徑，用於 API 操作）。

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 登入後清單在 3 秒內完成載入並顯示正確的檔案列表。
- **SC-002**: 「複製連結」操作在點擊後 500ms 內完成並顯示成功回饋。
- **SC-003**: 刪除操作從點擊確認到清單移除該項目，不超過 3 秒（含 API 回應時間）。
- **SC-004**: 清單顯示的 URL 與實際 GitLab Pages 可存取的 URL 100% 一致，無錯誤連結。
- **SC-005**: 刪除確認流程防止誤刪，用戶需要明確的二次確認動作才能完成刪除。
