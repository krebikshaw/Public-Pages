# Data Model: 上傳頁已發布檔案管理清單

**Branch**: `002-upload-file-manager` | **Date**: 2026-04-16  
**Phase**: 1 — 設計與資料模型

---

## 概述

本功能無後端資料庫，所有資料來自 GitLab API 即時查詢。資料模型描述的是 **前端執行期記憶體狀態**，即 JavaScript 物件的欄位與生命週期。

---

## Entity: PublishedFile

代表一個已發布的 HTML 頁面項目（從 GitLab Trees API 回應轉換而來）。

| 欄位 | 型別 | 來源 | 說明 |
|------|------|------|------|
| `filename` | `string` | Trees API `name` | 檔名，如 `demo.html` |
| `pageUrl` | `string` | 組合計算 | `{PAGES_BASE_URL}/{username}/{filename}`，不含 query string |
| `filePath` | `string` | Trees API `path` | repo 內相對路徑，如 `public/alice/demo.html`，作為 DELETE API 的 path 參數 |
| `encodedFilePath` | `string` | 組合計算 | `encodeURIComponent(filePath)`，URL-encoded 版本供 API 呼叫使用 |

**衍生規則**：
- `pageUrl = CONFIG.PAGES_BASE_URL + '/' + currentUser.username + '/' + filename`
- `encodedFilePath = encodeURIComponent('public/' + currentUser.username + '/' + filename)`

**驗證規則**（列入清單前過濾）：
- `type === 'blob'`（排除子目錄）
- `name.endsWith('.html')`（排除非 HTML 檔案）
- `name` 不含路徑穿越字符（`..`、`/`）

---

## State：FileListService 管理的執行期狀態

```javascript
// FileListService 內部狀態
{
  files: PublishedFile[],   // 當前清單，最多 20 筆
  isLoading: boolean,       // 是否正在載入清單
  error: string | null,     // 載入失敗訊息；null 表示正常
  deletingPath: string | null  // 目前正在刪除中的 filePath；null 表示無進行中的刪除
}
```

**狀態轉換**：

```
初始: { files: [], isLoading: false, error: null, deletingPath: null }

載入中: isLoading = true → API 回應 → isLoading = false
  ├─ 成功: files = [PublishedFile, ...], error = null
  └─ 失敗: files = [], error = "無法載入清單，請重新整理頁面"

刪除中: deletingPath = filePath → API 完成 → deletingPath = null
  ├─ 成功: files = files.filter(f => f.filePath !== filePath)
  └─ 失敗: 顯示錯誤提示，files 維持不變
```

---

## DOM 結構對應

```html
<!-- 清單區塊（登入後顯示） -->
<section id="section-file-list">
  <div id="file-list-loading">載入中…</div>          <!-- isLoading = true 時顯示 -->
  <div id="file-list-empty">尚無已發布的頁面</div>    <!-- files.length = 0 時顯示 -->
  <div id="file-list-error"></div>                   <!-- error != null 時顯示 -->
  <div id="file-list-container">                     <!-- max-height:400px; overflow-y:auto -->
    <!-- 每列對應一個 PublishedFile -->
    <div class="file-list-row" data-file-path="{filePath}">
      <span class="file-list-name">{filename}</span>
      <span class="file-list-url">{pageUrl}</span>
      <button class="btn-open">開啟</button>
      <button class="btn-copy">複製連結</button>
      <button class="btn-delete">刪除</button>
    </div>
  </div>
</section>

<!-- 刪除確認對話框（共用，動態填入檔名） -->
<div id="delete-confirm-modal" class="modal hidden">
  <div class="modal-body">
    <p id="delete-confirm-message"></p>
    <button id="btn-delete-confirm">確認刪除</button>
    <button id="btn-delete-cancel">取消</button>
  </div>
</div>
```
