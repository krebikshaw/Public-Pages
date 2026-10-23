# Quickstart: 上傳頁已發布檔案管理清單

**Branch**: `002-upload-file-manager` | **Date**: 2026-04-16

---

## 前置條件

- 已完成 spec 001（`public/publisher/index.html` 已部署於 GitLab Pages）
- 擁有對 `nbu/public-pages` 專案 Developer 以上權限的 GitLab 帳號
- 現代瀏覽器（Chrome 94+、Firefox 93+、Safari 15+）
- 已有至少一個 HTML 檔案透過 publisher 發布（用於測試清單顯示）

---

## 開發環境

本功能為純靜態頁面，無需任何建置工具或本機 server。

```bash
# 直接用瀏覽器開啟本機檔案（token CORS 限制，僅供 UI layout 視覺確認）
open public/publisher/index.html

# 或透過 GitLab Pages 的部署版本測試完整功能
# URL: https://public-pages-56dd79.fox.25sprout.com/publisher/
```

> **注意**：OAuth PKCE 流程需要實際的 `redirect_uri`（必須是 HTTPS 或 localhost）。  
> 功能整合測試須 commit 至 repo 並等待 pipeline 部署後在線上版本測試。

---

## 實作步驟

### Step 1：新增清單 HTML 區塊

在 `public/publisher/index.html` 的 `#section-upload`（上傳表單區塊）下方、`#section-pipeline`（pipeline 狀態區塊）上方，插入已發布頁面清單區塊：

```html
<!-- 已發布頁面清單（登入後顯示） -->
<section id="section-file-list" class="section hidden">
  <div class="card">
    <h2>我的已發布頁面</h2>
    <div id="file-list-loading" class="hidden">載入中…</div>
    <div id="file-list-empty" class="hidden">尚無已發布的頁面</div>
    <div id="file-list-error" class="error-text hidden"></div>
    <div id="file-list-container"></div>
  </div>
</section>

<!-- 刪除確認對話框 -->
<div id="delete-confirm-modal" class="modal hidden">
  <div class="modal-backdrop"></div>
  <div class="modal-body card">
    <p id="delete-confirm-message"></p>
    <div class="modal-actions">
      <button id="btn-delete-confirm" class="btn btn--danger">確認刪除</button>
      <button id="btn-delete-cancel" class="btn btn--secondary">取消</button>
    </div>
  </div>
</div>
```

### Step 2：新增 CSS 樣式

在 `<style>` 區塊新增：

```css
/* 已發布頁面清單 */
#file-list-container { max-height: 400px; overflow-y: auto; }
.file-list-row { display: flex; align-items: center; gap: 8px; padding: 8px 0; border-bottom: 1px solid var(--border); }
.file-list-name { font-weight: 600; min-width: 160px; }
.file-list-url  { flex: 1; color: var(--muted); font-size: 0.85em; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.btn-icon       { background: none; border: 1px solid var(--border); border-radius: 4px; padding: 4px 8px; cursor: pointer; font-size: 0.8em; }
.btn-icon:hover { border-color: var(--accent); color: var(--accent); }
.btn-icon:disabled { opacity: 0.4; cursor: not-allowed; }
.btn--danger    { background: #e53e3e; color: #fff; }
/* 刪除確認 modal */
.modal          { position: fixed; inset: 0; display: flex; align-items: center; justify-content: center; z-index: 100; }
.modal-backdrop { position: absolute; inset: 0; background: rgba(0,0,0,0.5); }
.modal .card    { position: relative; z-index: 1; min-width: 320px; }
.modal-actions  { display: flex; gap: 8px; justify-content: flex-end; margin-top: 16px; }
```

### Step 3：實作 FileListService

在現有 JS 物件定義區（`ApiService`、`UiService` 之後）新增：

```javascript
const FileListService = {
  // 載入清單
  async load() {
    const username = currentUser?.username;
    if (!username) return;
    UiService.setFileListLoading(true);
    try {
      const items = await ApiService.listFiles(username);
      const files = items
        .filter(i => i.type === 'blob' && i.name.endsWith('.html'))
        .slice(0, 20)
        .map(i => ({
          filename: i.name,
          filePath: i.path,
          encodedFilePath: encodeURIComponent(i.path),
          pageUrl: `${CONFIG.PAGES_BASE_URL}/${username}/${i.name}`
        }));
      UiService.renderFileList(files);
    } catch (err) {
      UiService.showFileListError('無法載入清單，請重新整理頁面');
    } finally {
      UiService.setFileListLoading(false);
    }
  },

  // 複製連結
  async copyUrl(pageUrl) {
    try {
      await navigator.clipboard.writeText(pageUrl);
      return true;
    } catch {
      // fallback: textarea + execCommand
      const ta = document.createElement('textarea');
      ta.value = pageUrl;
      ta.style.position = 'fixed'; ta.style.opacity = '0';
      document.body.appendChild(ta);
      ta.select();
      const ok = document.execCommand('copy');
      document.body.removeChild(ta);
      return ok;
    }
  },

  // 刪除流程
  async deleteFile(filePath, filename) {
    UiService.setRowDeleting(filePath, true);
    try {
      await ApiService.deleteFile(filePath, filename);
      await ApiService.triggerPipeline();
      UiService.removeFileRow(filePath);
      UiService.showToast(`已成功刪除 ${filename}`);
    } catch (err) {
      UiService.showToast(err.message || '刪除失敗，請稍後再試', 'error');
      UiService.setRowDeleting(filePath, false);
    }
  }
};
```

### Step 4：擴充 ApiService

在現有 `ApiService` 中新增三個方法：

```javascript
// 列出目錄下的 HTML 檔案（Trees API）
async listFiles(username) {
  const path = encodeURIComponent(`public/${username}`);
  const res = await this.request(`/api/v4/projects/${CONFIG.PROJECT_PATH}/repository/tree?path=${path}&ref=main&per_page=100`);
  if (res.status === 404) return [];  // 目錄不存在 = 無已發布頁面
  if (!res.ok) throw new Error('API 錯誤');
  return res.json();
},

// 刪除檔案（Repository Files DELETE）
async deleteFile(filePath, filename) {
  const encoded = encodeURIComponent(filePath);
  const res = await this.request(
    `/api/v4/projects/${CONFIG.PROJECT_PATH}/repository/files/${encoded}`,
    {
      method: 'DELETE',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ branch: 'main', commit_message: `remove: ${filePath}` })
    }
  );
  if (res.status === 401) { AuthService.handleExpiredToken(); return; }
  if (!res.ok) throw new Error(`刪除失敗（${res.status}）`);
},

// 觸發 pipeline
async triggerPipeline() {
  const res = await this.request(
    `/api/v4/projects/${CONFIG.PROJECT_PATH}/pipeline`,
    {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ ref: 'main' })
    }
  );
  if (!res.ok) console.warn('Pipeline 觸發失敗，但刪除操作已完成');
},
```

### Step 5：整合至登入流程與上傳完成後

1. **登入成功後**：在 `AuthService` 的登入後處理中加入 `FileListService.load()`
2. **部署成功後**：在 pipeline success handler 中加入 `FileListService.load()`（重新整理清單）
3. **登出時**：隱藏 `#section-file-list`

---

## 驗收測試清單

| # | 操作 | 預期結果 |
|---|------|---------|
| 1 | 登入後查看頁面 | `#section-file-list` 出現，顯示已發布的 HTML 檔案列表 |
| 2 | 尚無已發布頁面時 | 顯示「尚無已發布的頁面」提示文字 |
| 3 | 點擊「開啟」 | 新分頁開啟對應的 GitLab Pages URL |
| 4 | 點擊「複製連結」 | 剪貼簿含正確 URL，按鈕短暫顯示「已複製！」 |
| 5 | 點擊「刪除」 | 出現確認對話框，顯示正確檔名 |
| 6 | 確認刪除 | 列表移除該項目，顯示成功提示 |
| 7 | 取消刪除 | 對話框關閉，列表不變 |
| 8 | 上傳新檔案部署完成後 | 清單自動刷新並顯示新項目 |
| 9 | 登出後 | 清單區塊不顯示 |
