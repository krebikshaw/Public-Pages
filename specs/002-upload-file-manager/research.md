# Research: 上傳頁已發布檔案管理清單

**Branch**: `002-upload-file-manager` | **Date**: 2026-04-16  
**Phase**: 0 — 技術未知項調查

---

## Decision 1: 列出已發布檔案的 API

**Decision**: 使用 GitLab Repository **Trees API** 列出 `public/{username}/` 目錄下的所有 HTML 檔案。

**API 端點**:
```
GET /api/v4/projects/{id}/repository/tree?path=public%2F{username}&ref=main&per_page=100
```

回應範例：
```json
[
  { "id": "abc123", "name": "demo.html", "type": "blob", "path": "public/alice/demo.html", "mode": "100644" },
  { "id": "def456", "name": "report.html", "type": "blob", "path": "public/alice/report.html", "mode": "100644" }
]
```

前端過濾：僅顯示 `type === "blob"` 且 `name.endsWith(".html")` 的項目。

**Rationale**:
- Trees API 原生支援列目錄，回應格式簡潔，無需解析詳細 file metadata
- 同一 `ApiService` 已有完整的 Bearer token Fetch 封裝，直接複用
- `per_page=100` 涵蓋所有實際使用情境（每用戶不超過 20 筆顯示上限）

**Alternatives considered**:
- **Repository Files API (GET 單一檔案)**：僅能取得個別檔案，無法批次列舉，不適合
- **Commits / Diff API**：複雜度高，無法直接取得目錄內容

---

## Decision 2: 刪除檔案 + 觸發 Pipeline

**Decision**: 刪除分兩步驟：
1. `DELETE /api/v4/projects/{id}/repository/files/{file_path}` 刪除檔案（需 commit message）
2. `POST /api/v4/projects/{id}/pipeline` 手動觸發 pipeline，確保 GitLab Pages 重新部署，讓被刪除檔案的 URL 失效

**API 端點**:
```
# Step 1: 刪除檔案
DELETE /api/v4/projects/{id}/repository/files/{encoded_file_path}
Body: { branch: "main", commit_message: "remove: {username}/{filename}" }

# Step 2: 觸發 pipeline
POST /api/v4/projects/{id}/pipeline
Body: { ref: "main" }
```

**Rationale**:
- Repository Files DELETE API 需提供 commit message，自動建立 commit 記錄，符合 GitLab 操作習慣
- 刪除後主動觸發 pipeline 才能讓 Pages CDN 更新，純刪除 commit 本身不會即時令 URL 失效（GitLab Pages 由每次 pipeline 部署），用戶需求明確要求此行為
- 與 spec 001 commit flow 共用同一 `ApiService`，架構一致

**Alternatives considered**:
- **只刪除不觸發 pipeline**：URL 不會失效，不符合需求
- **使用 Commits API 建立「刪除 tree」的 commit**：API 複雜度更高，功能等效但無優勢

---

## Decision 3: 複製連結 — Clipboard API + Fallback

**Decision**: 優先使用 `navigator.clipboard.writeText()` 寫入剪貼簿；若失敗（`Promise reject` 或 `navigator.clipboard` 不存在），fallback 為建立隱藏的 `<textarea>` 元素 + `document.execCommand('copy')`（舊瀏覽器相容）；若仍失敗，則在頁面顯示 URL 文字框供用戶手動複製（最終 fallback）。

**Rationale**:
- Clipboard API 需要 HTTPS 或 localhost，GitLab Pages 已為 HTTPS，一般情況下可正常使用
- 企業內網環境偶有 CSP 或 Permission Policy 限制 clipboard-write，需提供 fallback
- `execCommand('copy')` 雖已廢棄但瀏覽器實際上仍廣泛支援，作為第二層保障
- 最終 fallback 的文字框確保用戶永遠有辦法取得連結，不靜默失敗

**Alternatives considered**:
- **只用 Clipboard API，失敗靜默**：不符合 FR-005（失敗需告知用戶）
- **只用 execCommand**：已廢棄，不應作為主要方案

---

## Decision 4: 清單 UI — 捲動容器（不分頁）

**Decision**: 清單區塊使用固定最大高度容器（`max-height: 400px; overflow-y: auto`），顯示最多 20 筆，超過時在同一容器內捲動，不實作分頁控制元件。

**Rationale**:
- 用戶確認上限為 20 筆，單一捲動容器實作最簡單、符合 YAGNI
- 分頁需要額外的 state 管理（currentPage、pageSize）和 UI 元件，對 20 筆資料過度工程化
- 捲動體驗在十數位內部用戶的實際使用情境中已足夠

**Alternatives considered**:
- **無限制顯示**：不符合 FR-010（上限 20 筆）
- **分頁控制元件**：用戶未要求，超出 YAGNI 範圍

---

## Decision 5: 「刪除中」UI 狀態設計

**Decision**: 點擊「確認刪除」後，該列的三個按鈕全部 `disabled`，顯示「刪除中…」提示；刪除+觸發 pipeline 完成後，直接從 DOM 移除該列（不重新呼叫 Trees API）。

**Rationale**:
- 防止用戶在操作進行中重複點擊（同 spec 001 的 publish 按鈕防重複提交原則）
- 直接更新 DOM 比重新 fetch 整個清單更快，且不增加不必要的 API 呼叫
- Pipeline 觸發後不在清單頁輪詢 pipeline 狀態（刪除動作的 pipeline 監控超出本功能範圍）
