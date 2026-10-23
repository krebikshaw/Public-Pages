# Implementation Plan: HTML Page Publisher

**Branch**: `001-html-page-publisher` | **Date**: 2026-04-02 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-html-page-publisher/spec.md`

## Summary

建立一個純靜態前端頁面（`public/publisher/index.html`），部署於 `nbu/public-pages` GitLab Pages Unique Domain（`https://public-pages-56dd79.fox.25sprout.com/publisher/`）。使用者透過 OAuth 2.0 PKCE 流程以 GitLab 帳號登入，選取本機 HTML 檔案並指定檔案名稱，系統透過 GitLab Repository Files API 將檔案 commit 至 `public/{username}/{filename}.html`，觸發 CI/CD pipeline，並在頁面上即時輪詢 pipeline 狀態，成功後顯示部署網址 `https://nbu.fox.25sprout.com/public-pages/{username}/{filename}.html?v={hash}`。`?v={hash}` 為 4 字元 SHA-256 cache-busting 參數，在選取檔案時即時計算並顯示於預覽網址；pipeline 成功後顯示 10 秒 CDN 生效倒數再釋出可點擊連結。

## Technical Context

**Language/Version**: Vanilla JavaScript (ES2020+), HTML5, CSS3  
**Primary Dependencies**: 瀏覽器原生 API（Web Crypto API、Fetch API、sessionStorage）；無第三方 npm 套件  
**Storage**: `sessionStorage`（access token、pipeline ID），無後端資料庫  
**Testing**: 手動瀏覽器測試 + GitLab pipeline 整合測試（MVP 階段不建立自動化測試）  
**Target Platform**: 現代瀏覽器（Chrome 94+、Firefox 93+、Safari 15+），靜態 GitLab Pages  
**Project Type**: 單頁靜態 web 應用，無建置步驟  
**Performance Goals**: OAuth 流程 < 5 秒；GitLab API commit < 3 秒；pipeline 輪詢間隔 ≤ 10 秒  
**Constraints**: 無後端 server；所有邏輯於瀏覽器執行；token 不離開 sessionStorage；不引入任何 npm 建置工具  
**Scale/Scope**: 單頁工具，預計十數位內部用戶；pipeline 逾時上限 5 分鐘

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| 原則 | 適用性 | 狀態 | 說明 |
|------|--------|------|------|
| I. 模組化架構（Backstage Module Pattern） | ❌ 不適用 | PASS | 本功能為獨立靜態頁面，非 Backstage Laravel 模組 |
| II. 分層設計（Repository / Service / Controller） | ❌ 不適用 | PASS | 無後端 server，純瀏覽器端邏輯 |
| III. 安全性優先 | ✅ 適用 | PASS | OAuth PKCE（無 client_secret）、token 存於 sessionStorage、路徑注入防護（FR-005、FR-006） |
| IV. 測試紀律 | ⚠️ 部分適用 | PASS（MVP 豁免） | MVP 階段僅手動測試；自動化測試列為後續改善項目 |
| V. 簡潔與一致性 | ✅ 適用 | PASS | 無 npm 套件依賴、YAGNI、繁體中文註解、單一 HTML 檔案 |

> **結論**：無憲法違規，可進入 Phase 0。

## Project Structure

### Documentation (this feature)

```text
specs/001-html-page-publisher/
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
├── publisher/
│   └── index.html       # Publisher UI — 單一自含式 HTML/CSS/JS 檔案
├── {username}/          # 用戶上傳後由 pipeline 部署生成
│   └── {filename}.html
└── index.html           # 已存在：NBU Pages 入口首頁
```

**Structure Decision**: 採用 Option 1（單一靜態檔案）。所有邏輯、樣式、腳本內嵌於 `public/publisher/index.html`，無建置步驟，符合 GitLab Pages 純靜態部署限制。用戶上傳的檔案透過 GitLab API commit 至 `public/{username}/{filename}.html`，由 CI/CD 一併部署。
