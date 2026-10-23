<!--
Sync Impact Report
- 版本變更: 1.0.0 → 1.1.0 (平台升級 PHP 8.4 / Laravel 12 / MySQL 8.4)
- 修改章節:
  - 技術棧規範（後端）：PHP 7.1→8.4、Laravel 6→12、MySQL 新增 8.4、jwt-auth 1.0→2.3、bouncer rc.6→1.0.3、L5-Swagger 6→8.6、laravel-auditing 9→14
  - 核心原則 IV.測試紀律：PHPUnit 8→11
- 後續 TODO: 無

---
前次 Sync Impact Report
- 版本變更: 0.0.0 → 1.0.0 (首次制定)
- 新增原則:
  - I. 模組化架構（Backstage Module Pattern）
  - II. 分層設計（Repository / Service / Controller）
  - III. 安全性優先
  - IV. 測試紀律
  - V. 簡潔與一致性
- 新增章節:
  - 技術棧規範
  - 開發工作流程
  - Governance
- 刪除章節: 無（首次制定）
- 範本一致性檢查:
  - .specify/templates/plan-template.md ✅ 無需更新（通用範本）
  - .specify/templates/spec-template.md ✅ 無需更新（通用範本）
  - .specify/templates/tasks-template.md ✅ 無需更新（通用範本）
- 後續 TODO: 無
-->

# SurveyCake Audience Backstage 專案憲法

## 核心原則

### I. 模組化架構（Backstage Module Pattern）

本專案採用**設定驅動的動態 CRUD 引擎**作為核心架構模式，
所有新功能 MUST 遵循此模式：

- 每個業務實體透過 Model 的 `$config` 陣列宣告其 CRUD 行為
- `BackStage.php` 根據 URL 路由動態解析 Model，
  委派至對應的 Module 類別（`VueForm`、`VueList`、`VueSearch` 等）
- 新增模組 MUST 繼承 `AbstractModule` 或 `AbstractModuleList`
- 模型特定的覆寫邏輯 MUST 放置於 `app/Backstage/Modules/Extend/` 目錄
- 單一控制器 `AdminController` 服務所有 CRUD 端點，
  禁止為每個實體新建獨立控制器（除非有明確架構理由並經審查）
- 新增模組 MUST 建立選單項目（`menu_items`）與角色權限（`abilities`）
  的 migration，透過呼叫 `SetAdminRoleSeeder` 自動賦權。
  若未註冊，模組不會出現在後台側邊欄，角色也無法存取。
  具體步驟：(1) 在 `AbilityMenuSeeder` 加入 `createMenuItem()` 定義
  (2) 建立 migration 冪等插入 `menu_items` 並呼叫 `SetAdminRoleSeeder`

### II. 分層設計（Repository / Service / Controller）

程式碼 MUST 遵循三層分離原則：

- **Controller 層**：僅處理 HTTP 請求/回應轉換與路由邏輯
- **Service 層**（`app/Services/`）：封裝業務邏輯，
  可注入多個 Repository
- **Repository 層**（`app/Repositories/`）：封裝資料存取邏輯，
  MUST 實作對應的 Interface（`Repositories/Interfaces/`）
- **Model 層**（`app/Models/`）：所有 Model MUST 繼承
  `AbstractModel`，提供統一的 `creator()` 關聯與 `creator_name` 存取器
- 禁止在 Controller 中直接操作 Eloquent 查詢

### III. 安全性優先

- 認證 MUST 使用 JWT（`tymon/jwt-auth`），
  分為 `api` guard（AdminUser）與 `webapi` guard（User）
- 授權 MUST 使用 Bouncer RBAC（`silber/bouncer`），
  透過 `AbilityCheck` 中介層強制執行 `PERMISSION_VIEW` / `PERMISSION_EDIT`
- 所有 API 端點 MUST 經過 `BackstageAuth` 中介層驗證
- 社群登入（Google / Facebook）MUST 透過 Laravel Socialite 處理
- CSP（Content Security Policy）設定 MUST 維護於 `config/csp.php`
- IP 存取控制透過 `IPAuthService` 管理

### IV. 測試紀律

- 單元測試使用 PHPUnit 11.x，測試案例 MUST 放置於
  `tests/Unit/` 或 `tests/Feature/`
- 測試命名規則：檔名使用 `{程式名}Test`（如 `CakeServiceTest`），
  方法使用 `test{Method名}`（如 `testGetMemberInfo`）
- 瀏覽器測試使用 Laravel Dusk（Headless Chrome）
- 測試環境 MUST 使用獨立的 `.env.testing` 設定檔
  與獨立的測試資料庫
- Mock 使用 Mockery，假資料使用 Faker

### V. 簡潔與一致性

- 遵循 YAGNI 原則：不為尚未確認的需求預先實作功能
- 前後端程式碼中的註解 MUST 使用繁體中文
- 前端元件 MUST 依循 Atomic Design 模式
  （`atoms/`、`molecules/`、`organisms/`）
- 優先使用 25sprout 內部元件庫（`@25sprout-internal/wc-*`）
  與 Element UI，避免引入重複功能的第三方套件
- Model 稽核追蹤 MUST 透過 `owen-it/laravel-auditing` 記錄

## 技術棧規範

### 後端

| 項目 | 規範 |
|---|---|
| 語言 | PHP >= 8.4 |
| 框架 | Laravel 12.x |
| 資料庫 | MySQL 8.4（utf8mb4 / utf8mb4_general_ci），Redis 用於快取 |
| 認證 | JWT（tymon/jwt-auth 2.3.x） |
| 授權 | Bouncer（silber/bouncer v1.0.3） |
| API 文件 | L5-Swagger 8.6.x + zircote/swagger-php 3.x |
| 檔案儲存 | AWS S3、Google Cloud Storage、MinIO（開發環境） |
| 佇列 | 支援 sync / database / SQS / Redis |
| Excel 匯出 | maatwebsite/excel 3.x |
| 圖片處理 | intervention/image 2.x |
| 階層資料 | franzose/closure-table 5.x（Unit 組織樹） |
| HTTP 客戶端 | Guzzle 6.x |
| 稽核日誌 | owen-it/laravel-auditing 14.x |

### 前端

| 項目 | 規範 |
|---|---|
| 框架 | Vue.js 2.5+（SPA 架構） |
| 狀態管理 | Vuex 3.x |
| 路由 | Vue Router 3.x |
| UI 元件庫 | Element UI 2.x + @25sprout-internal/wc-* 0.2.x |
| 圖示 | Font Awesome Pro 5.x（vue-fontawesome） |
| 富文本編輯 | Quill（vue-quill-editor） |
| HTTP 客戶端 | axios + vue-resource |
| 建置工具 | Laravel Mix 0.12 + Webpack（自訂 webpack.config.js） |
| CSS 預處理 | SASS（Bootstrap SASS 3.x） |
| 版本管理 | standard-version（semver + CHANGELOG） |

### 基礎設施

| 項目 | 規範 |
|---|---|
| 容器化 | Laradock（Docker） |
| 雲端服務 | AWS（S3、SQS、SES、SNS）、Google Cloud Storage |
| 反向代理 | fideloper/proxy（負載平衡器後方運行） |
| 時區 | Asia/Taipei |

## 開發工作流程

### 分支與提交

- 功能分支 MUST 從主分支建立，命名格式：`feature/{功能描述}`
- Commit 訊息 MUST 遵循 Conventional Commits 格式
- 使用 `standard-version` 管理版本號與 CHANGELOG

### 新增功能流程

1. 若為新業務實體：建立 Model（繼承 `AbstractModel`）→
   設定 `$config` → 系統自動產生 CRUD
2. 若需客製邏輯：於 `Modules/Extend/` 新增覆寫類別
3. 若需獨立業務邏輯：建立 Service → 注入 Repository
4. API 端點 MUST 在 Swagger 文件中記錄

### 環境管理

- 開發環境使用 `.env` + Laradock
- 測試環境使用 `.env.testing` + 獨立測試資料庫
- 敏感設定（API 金鑰、密鑰）MUST 透過環境變數管理，
  禁止寫死於程式碼中

## Governance

本憲法為專案最高指導原則，所有開發實踐 MUST 遵循本文件：

- 修訂本憲法 MUST 提供：變更說明、修改理由、遷移計畫
- 所有 PR / Code Review MUST 驗證是否符合憲法原則
- 新增複雜度 MUST 附帶明確理由
- 版本號遵循語意化版本：
  - MAJOR：不相容的原則變更或移除
  - MINOR：新增原則或章節、重大指引擴充
  - PATCH：措辭修正、格式調整、非語意性修改

**Version**: 1.1.0 | **Ratified**: 2026-02-13 | **Last Amended**: 2026-03-12
