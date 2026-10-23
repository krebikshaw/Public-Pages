# Public Pages

把 HTML 放進 `public/`，push 到 `main` 後會由 GitHub Actions 自動部署到 GitHub Pages。

**網站網址**：<https://krebikshaw.github.io/Public-Pages/>

## 使用方式

1. 把 HTML 檔放到 `public/` 底下，例如 `public/my-report.html`。
2. commit 並 push 到 `main`。
3. 約一兩分鐘後，即可透過下列網址存取：

```
https://krebikshaw.github.io/Public-Pages/my-report.html
```

放在子資料夾的檔案，網址就包含該路徑，例如 `public/team/a.html` → `/Public-Pages/team/a.html`。

## 注意

- 這個 repo 與 Pages 網址是公開的，**不要放機密內容**。
- 部署設定在 [.github/workflows/pages.yml](.github/workflows/pages.yml)；Settings → Pages 的 Source 需設為 **GitHub Actions**。
- 同名檔案再次 push 會直接覆蓋舊版本。
