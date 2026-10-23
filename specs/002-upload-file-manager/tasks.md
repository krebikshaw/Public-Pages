# Tasks: 上傳頁已發布檔案管理清單

**Input**: Design documents from `/specs/002-upload-file-manager/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅, quickstart.md ✅

**Target File**: `public/publisher/index.html`（唯一修改目標）

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different sections of the same file, no mutual dependencies)
- **[Story]**: Which user story this task belongs to (US1–US4)
- All tasks modify `public/publisher/index.html` unless stated otherwise

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 確認現有程式結構，建立本次擴充的骨架

- [x] T001 閱讀並理解 `public/publisher/index.html` 現有架構：CONFIG 常數、ApiService、UiService、AuthService、及各 section 的顯示/隱藏邏輯
- [x] T002 在 `public/publisher/index.html` 的 `<style>` 區塊末尾新增已發布清單與 modal 的 CSS 樣式（`.file-list-row`、`#file-list-container`、`.modal`、`.btn-icon` 等，參考 `quickstart.md` Step 2）
- [x] T003 在 `public/publisher/index.html` 的 HTML body 中插入 `#section-file-list` 區塊與 `#delete-confirm-modal` 對話框骨架（參考 `quickstart.md` Step 1，置於 `#section-upload` 與 `#section-pipeline` 之間）

**Checkpoint**: 清單區塊的 HTML/CSS 骨架已就位（此時顯示為空區塊）

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 新增所有 User Story 共用的 API 方法與 FileListService 骨架

**⚠️ CRITICAL**: 下列任務完成前，任何 User Story 的功能皆無法運作

- [x] T004 在 `public/publisher/index.html` 的 `ApiService` 物件中新增 `listFiles(username)` 方法：呼叫 GitLab Trees API `GET /api/v4/projects/{PROJECT_PATH}/repository/tree?path=public%2F{username}&ref=main&per_page=100`，404 回傳空陣列，其他錯誤拋出例外（參考 `contracts/gitlab-api.md` §1）
- [x] T005 [P] 在 `public/publisher/index.html` 的 `ApiService` 物件中新增 `deleteFile(filePath)` 方法：呼叫 `DELETE /api/v4/projects/{PROJECT_PATH}/repository/files/{encodedPath}`，處理 401（觸發 token 過期流程）、404（視為已刪除）、其他錯誤拋出（參考 `contracts/gitlab-api.md` §2）
- [x] T006 [P] 在 `public/publisher/index.html` 的 `ApiService` 物件中新增 `triggerPipeline()` 方法：呼叫 `POST /api/v4/projects/{PROJECT_PATH}/pipeline` with `{ ref: 'main' }`，失敗只 `console.warn` 不拋出（參考 `contracts/gitlab-api.md` §3）
- [x] T007 在 `public/publisher/index.html` 中定義 `FileListService` 物件骨架，包含 `load()`、`copyUrl(pageUrl)`、`deleteFile(filePath, filename)` 三個空方法（方法體留 TODO，後續 Phase 填入）

**Checkpoint**: ApiService 的三個新方法與 FileListService 骨架已就位

---

## Phase 3: User Story 1 - 查看已發布的頁面清單 (Priority: P1) 🎯 MVP

**Goal**: 登入後清單區塊顯示用戶在 `public/{username}/` 下的所有 HTML 檔案，含空清單提示與錯誤狀態

**Independent Test**: 登入後確認 `#section-file-list` 出現，清單內容與 GitLab repo 實際目錄一致；尚無檔案時顯示「尚無已發布的頁面」

### Implementation for User Story 1

- [x] T008 [US1] 實作 `FileListService.load()` 方法體：呼叫 `ApiService.listFiles(username)`，過濾 `type === 'blob' && name.endsWith('.html')`，`slice(0, 20)`，組合 `PublishedFile` 物件（`filename`、`filePath`、`encodedFilePath`、`pageUrl`），呼叫 `UiService.renderFileList(files)` 在 `public/publisher/index.html`
- [x] T009 [US1] 在 `UiService` 中實作 `renderFileList(files)` 方法：清空 `#file-list-container`，若 files 為空則顯示 `#file-list-empty`，否則逐列建立 `.file-list-row` DOM（含 `data-file-path`、檔名 span、URL span、三個按鈕），超過容器高度時可捲動 在 `public/publisher/index.html`
- [x] T010 [US1] 在 `UiService` 中實作 `setFileListLoading(isLoading)` 方法：顯示/隱藏 `#file-list-loading`，loading 時隱藏 `#file-list-container` 與 `#file-list-empty` 在 `public/publisher/index.html`
- [x] T011 [US1] 在 `UiService` 中實作 `showFileListError(msg)` 方法：在 `#file-list-error` 顯示錯誤訊息 在 `public/publisher/index.html`
- [x] T012 [US1] 在登入成功後的 handler（`AuthService` 或登入後 callback）中呼叫 `FileListService.load()`，在 `SectionService.showSection('upload')` 後執行；確保 `#section-file-list` 在登入時顯示、登出時隱藏 在 `public/publisher/index.html`

**Checkpoint**: 登入後清單區塊顯示正確檔案列表（或空提示）；網路錯誤時顯示錯誤訊息

---

## Phase 4: User Story 2 - 點擊開新視窗預覽頁面 (Priority: P2)

**Goal**: 清單中每列的「開啟」按鈕點擊後以新分頁開啟對應的 GitLab Pages URL

**Independent Test**: 點擊任一列「開啟」按鈕，確認瀏覽器新分頁開啟 `{PAGES_BASE_URL}/{username}/{filename}`

### Implementation for User Story 2

- [x] T013 [US2] 在 `UiService.renderFileList()` 產生的「開啟」按鈕上綁定 click 事件：`window.open(pageUrl, '_blank', 'noopener,noreferrer')` 在 `public/publisher/index.html`

**Checkpoint**: 點擊「開啟」可在新分頁預覽對應頁面

---

## Phase 5: User Story 3 - 複製頁面連結 (Priority: P2)

**Goal**: 點擊「複製連結」將 pageUrl 複製至剪貼簿，顯示成功/失敗回饋

**Independent Test**: 點擊「複製連結」後貼入文字框，確認內容與清單 URL 完全一致；按鈕短暫顯示「已複製！」

### Implementation for User Story 3

- [x] T014 [US3] 實作 `FileListService.copyUrl(pageUrl)` 方法體：優先使用 `navigator.clipboard.writeText(pageUrl)`；失敗時 fallback 為隱藏 `<textarea>` + `document.execCommand('copy')`；兩者皆失敗時回傳 `false` 在 `public/publisher/index.html`
- [x] T015 [US3] 在 `UiService.renderFileList()` 產生的「複製連結」按鈕上綁定 click 事件：呼叫 `FileListService.copyUrl(pageUrl)`，成功時按鈕文字暫改為「已複製！」並於 1500ms 後恢復；失敗時在清單旁顯示 URL 文字框供手動複製 在 `public/publisher/index.html`

**Checkpoint**: 複製連結功能在 HTTPS 與各種瀏覽器 CSP 環境下均可使用

---

## Phase 6: User Story 4 - 刪除已發布的頁面 (Priority: P3)

**Goal**: 點擊「刪除」顯示確認 modal，確認後刪除檔案並觸發 pipeline，清單即時移除對應項目

**Independent Test**: 點擊「刪除」→ 確認 → 查詢 GitLab repo 確認檔案已不存在，清單中不再顯示；取消則清單不變

### Implementation for User Story 4

- [x] T016 [US4] 實作 `UiService.showDeleteConfirm(filename, onConfirm)` 方法：顯示 `#delete-confirm-modal`，填入「確定要刪除 {filename} 嗎？此操作無法復原。」，綁定「確認刪除」按鈕呼叫 `onConfirm()`、「取消」按鈕關閉 modal 在 `public/publisher/index.html`
- [x] T017 [US4] 實作 `FileListService.deleteFile(filePath, filename)` 方法體：呼叫 `ApiService.deleteFile(filePath)`，成功後呼叫 `ApiService.triggerPipeline()`，最後呼叫 `UiService.removeFileRow(filePath)` 與 `UiService.showToast('已成功刪除 ' + filename)`；任一步驟失敗依 `contracts/gitlab-api.md` §2 顯示對應錯誤訊息 在 `public/publisher/index.html`
- [x] T018 [US4] 在 `UiService` 中實作 `setRowDeleting(filePath, isDeleting)` 方法：將指定列的三個按鈕設為 `disabled`，並顯示「刪除中…」提示；`isDeleting = false` 時恢復 在 `public/publisher/index.html`
- [x] T019 [US4] 在 `UiService` 中實作 `removeFileRow(filePath)` 方法：從 DOM 找到 `data-file-path === filePath` 的 `.file-list-row` 並移除；若移除後 container 為空則顯示 `#file-list-empty` 在 `public/publisher/index.html`
- [x] T020 [US4] 在 `UiService.renderFileList()` 產生的「刪除」按鈕上綁定 click 事件：呼叫 `UiService.showDeleteConfirm(filename, () => { UiService.setRowDeleting(filePath, true); FileListService.deleteFile(filePath, filename); })` 在 `public/publisher/index.html`
- [x] T021 [US4] 在 `UiService` 中實作 `showToast(message, type='success')` 方法：在畫面顯示短暫浮動提示（3 秒後自動消失），`type='error'` 時使用紅色樣式 在 `public/publisher/index.html`

**Checkpoint**: 完整刪除流程可運作：確認 → 呼叫 API → 觸發 pipeline → 清單更新

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: 整合收尾、自動刷新、邊界處理

- [x] T022 在部署成功後的 pipeline success handler 中，於顯示成功訊息後呼叫 `FileListService.load()` 自動刷新清單（FR-008）在 `public/publisher/index.html`
- [x] T023 [P] 確認登出（AuthService logout）時隱藏 `#section-file-list` 並清空 `#file-list-container`（FR-010）在 `public/publisher/index.html`
- [x] T024 [P] 為長檔名（> 40 字元）加入 CSS `text-overflow: ellipsis` 防止破版，驗證排版在不同視窗寬度下正常在 `public/publisher/index.html`
- [x] T025 依 `quickstart.md` 驗收測試清單（9 個場景）逐項手動測試，確認全數通過

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: 無依賴，立即開始
- **Phase 2 (Foundational)**: 依賴 Phase 1 完成 — 阻擋所有 User Story
- **Phase 3 (US1)**: 依賴 Phase 2 完成
- **Phase 4 (US2)**: 依賴 Phase 3 完成（需要 `renderFileList` 已產生按鈕）
- **Phase 5 (US3)**: 依賴 Phase 3 完成（同上）
- **Phase 6 (US4)**: 依賴 Phase 3 完成（需要 `renderFileList` 已產生按鈕）
- **Phase 7 (Polish)**: 依賴所有 User Story 完成

### User Story Dependencies

- **US1 (P1)**: 依賴 Phase 2 — 無其他 Story 依賴
- **US2 (P2)**: 依賴 US1（需要 `renderFileList` 已產生「開啟」按鈕 DOM）
- **US3 (P2)**: 依賴 US1（需要 `renderFileList` 已產生「複製連結」按鈕 DOM）
- **US4 (P3)**: 依賴 US1（需要 `renderFileList` 已產生「刪除」按鈕 DOM）

> US2、US3、US4 彼此獨立，可於 US1 完成後並行開發

### Within Each User Story

- Service 方法實作 → UI 事件綁定（需 DOM 產生後才能綁定）
- T004~T006 可並行（Phase 2 內部）
- T013、T014+T015 可並行（各自獨立）
- T016~T021 需依序（modal → delete logic → UI helper → 事件綁定）

---

## Parallel Example: Phase 2

```
可同步進行：
- T005 deleteFile() API 方法
- T006 triggerPipeline() API 方法
（T004 listFiles 須先完成以便 FileListService 骨架使用）
```

## Parallel Example: US2 + US3（US1 完成後）

```
可同步進行：
- Phase 4 (US2): T013 開啟按鈕事件綁定
- Phase 5 (US3): T014 copyUrl 實作 + T015 複製按鈕事件綁定
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. 完成 Phase 1: Setup（HTML/CSS 骨架）
2. 完成 Phase 2: Foundational（API 方法 + FileListService 骨架）
3. 完成 Phase 3: US1（清單載入與顯示）
4. **STOP and VALIDATE**: 登入後確認清單正確顯示
5. 如驗證通過，繼續 Phase 4–6

### Full Delivery Order

Phase 1 → Phase 2 → Phase 3 → Phase 4 + Phase 5（並行）→ Phase 6 → Phase 7

---

## Summary

| 統計 | 數量 |
|------|------|
| 總任務數 | 25 |
| Phase 1 Setup | 3 |
| Phase 2 Foundational | 4 |
| US1 查看清單 (P1) | 5 |
| US2 開啟頁面 (P2) | 1 |
| US3 複製連結 (P2) | 2 |
| US4 刪除頁面 (P3) | 6 |
| Polish | 4 |
| 可並行任務 [P] | 8 |
| MVP 範圍（US1）| T001–T012 |
