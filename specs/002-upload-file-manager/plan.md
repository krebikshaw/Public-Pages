# Implementation Plan: 上傳頁已發布檔案管理清單

**Branch**: `002-upload-file-manager` | **Date**: 2026-04-16 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/002-upload-file-manager/spec.md`

## Summary

在現有 `public/publisher/index.html` 中新增「已發布頁面清單」區塊，透過 GitLab Repository Trees API 列出 `public/{username}/` 下的所有 HTML 檔案，提供開啟（新分頁）、複製連結（Clipboard API with fallback）、刪除（確認對話框 + Repository Files DELETE API + 觸發新 pipeline）等功能。清單最多顯示 20 筆，超過時在固定高度容器內捲動。整體為 Vanilla JS 擴充，無新增檔案，無建置步驟。

## Technical Context

**Language/Version**: Vanilla JavaScript (ES2020+), HTML5, CSS3（延續 spec 001 技術選型）  
**Primary Dependencies**: 瀏覽器原生 API（Clipboard API、Fetch API、sessionStorage）；無第三方 npm 套件  
**Storage**: `sessionStorage`（access token，延續現有 `gl_token`）；無後端資料庫  
**Testing**: 手動瀏覽器測試（延續 spec 001 MVP 豁免策略）  
**Target Platform**: 現代瀏覽器（Chrome 94+、Firefox 93+、Safari 15+），靜態 GitLab Pages  
**Project Type**: 單一靜態 HTML 檔案擴充，無建置步驟  
**Performance Goals**: 清單載入 < 3 秒；複製連結 < 500ms；刪除操作（含 API）< 3 秒  
**Constraints**: 無後端 server；所有邏輯在瀏覽器執行；不引入任何 npm 套件；須相容現有 publisher 程式架構  
**Scale/Scope**: 十數位內部用戶；每位用戶最多顯示 20 筆已發布頁面

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原則 | 適用性 | 狀態 | 說明 |
|------|--------|------|------|
| I. 模組化架構（Backstage Module Pattern） | ❌ 不適用 | PASS | 本功能為獨立靜態頁面，非 Backstage Laravel 模組 |
| II. 分層設計（Repository / Service / Controller） | ❌ 不適用 | PASS | 無後端 server，純瀏覽器端邏輯 |
| III. 安全性優先 | ✅ 適用 | PASS | token 仍存於 sessionStorage；刪除前強制二次確認防誤操作；路徑使用 currentUser.username，禁止用戶覆寫 |
| IV. 測試紀律 | ⚠️ 部分適用 | PASS（MVP 豁免） | 延續 spec 001 方針：MVP 階段僅手動測試 |
| V. 簡潔與一致性 | ✅ 適用 | PASS | 無新增 npm 依賴；擴充現有 HTML 檔而非新建；繁體中文註解；YAGNI |

> **結論**：無憲法違規，可進入 Phase 0。

## Project Structure

### Documentation (this feature)

```text
specs/002-upload-file-manager/
├── plan.md              # 本檔（/speckit.plan 輸出）
├── research.md          # Phase 0 輸出
├── data-model.md        # Phase 1 輸出
├── quickstart.md        # Phase 1 輸出
├── contracts/           # Phase 1 輸出
│   └── gitlab-api.md
└── tasks.md             # Phase 2 輸出（/speckit.tasks 產生）
```

### Source Code (repository root)

```text
public/
└── publisher/
    └── index.html       # 唯一修改目標：新增「已發布頁面清單」UI 區塊與對應 JS 邏輯
```

**Structure Decision**: 不新增任何檔案。所有 UI、CSS、JS 擴充均寫入現有的 `public/publisher/index.html`，維持零建置步驟的靜態頁面部署方式。新增的 JS 邏輯統一放置於現有 `ApiService` / `UiService` 物件模式中，新增 `FileListService` 物件處理清單相關操作。

## Complexity Tracking

> 無憲法違規，本節不填寫。
