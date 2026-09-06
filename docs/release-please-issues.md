# Release Please POC 問題紀錄

本文件記錄此 Monorepo Docker Image POC 在導入 Release Please Manifest Mode 時實際遇到的問題、原因與處理結果。

## POC 目前設定

- 使用 `googleapis/release-please-action@v4`
- 使用 Manifest Mode
- App A 路徑：`images/app-a`
- App B 路徑：`images/app-b`
- 版本 Tag 格式：`app-a-v1.1.0`、`app-b-v1.1.0`
- 版本獨立設定：`separate-pull-requests: true`
- 目前嘗試使用：`skip-github-pull-request: true`

## 問題一：GitHub Actions 無法建立 Release PR

### 錯誤訊息

```text
release-please failed: GitHub Actions is not permitted to create or approve pull requests.
```

### 原因

Workflow 雖然已宣告：

```yaml
permissions:
  contents: write
  issues: write
  pull-requests: write
```

但 Repository 層級仍禁止 `GITHUB_TOKEN` 建立或核准 Pull Request。Workflow 的 `pull-requests: write` 權限本身不足以開啟這個 Repository 設定。

### 處理方式

在 GitHub Repository 進入：

```text
Settings → Actions → General → Workflow permissions
```

開啟：

```text
Allow GitHub Actions to create and approve pull requests
```

官方文件：[Managing GitHub Actions settings for a repository](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/enabling-features-for-your-repository/managing-github-actions-settings-for-a-repository)

### 結果

設定開啟後，Actions 可以成功建立 Release PR。

## 問題二：初始化提交造成錯誤的 Minor 版本

### 現象

初始提交使用：

```text
feat: first init
```

該提交同時新增 App A 與 App B，因此 Release Please 將兩個 App 都判定為需要 Minor release：

```text
images/app-a: 1.0.0 → 1.1.0
images/app-b: 1.0.0 → 1.1.0
```

因此同時建立了兩個 Release PR。

### 原因

第一次發布時 Repository 尚未有各專案的 baseline Tag。Release Please 會從歷史提交開始解析 Conventional Commits，初始化用的 `feat:` 也被當成正式功能。

### 處理方式

在初始提交 `0ab160a` 建立 baseline Tags：

```text
app-a-v1.0.0
app-b-v1.0.0
```

之後 App A 的 `fix:` 變更才會從 `1.0.0` 正確計算為 `1.0.1`。

### 建議

初始化提交應使用不會觸發版本升級的提交類型，例如：

```text
chore: initialize release please POC
```

若 Repository 已經存在，則應在第一次測試前明確建立每個專案的 baseline Tag。

## 問題三：`include-component-in-tag` 設定被 Action input 覆蓋

### 現象

雖然 `release-please-config.json` 已設定：

```json
"include-component-in-tag": true
```

但 Actions log 顯示：

```text
include-component-in-tag: false
```

### 原因

`release-please-action` 的 Action input 預設值為 `false`。Action input 覆蓋了 manifest config 的設定，導致 Release Please 以不含 component 的方式尋找上一個 Tag，例如尋找 `v1.0.1`，而不是：

```text
app-a-v1.0.1
```

### 處理方式

在 workflow 的 `with` 區塊明確設定：

```yaml
with:
  manifest-file: .release-please-manifest.json
  config-file: release-please-config.json
  include-component-in-tag: true
```

### 結果

Actions log 已確認實際使用：

```text
include-component-in-tag: true
```

## 問題四：`skip-github-pull-request: true` 沒有產生 Tag

### 目的

希望在功能 PR 合併到 `main` 後：

```text
feat commit → Actions → 直接建立 app-a-v1.1.0 Tag
```

不需要再合併 Release PR。

### 實際現象

Actions 顯示成功，且確認 input 已生效：

```text
skip-github-pull-request: true
include-component-in-tag: true
```

但是 log 在以下階段後結束：

```text
Building releases
images/app-a: simple
images/app-b: simple
```

沒有建立 GitHub Release，也沒有產生 Tag。由於 `releases_created` 為 `false`，POC evidence step 也會被跳過，因此不會產生 Artifact。

### 實際驗證結果

App A 的 `feat` PR 已合併，但 Tags 仍停留在：

```text
app-a-v1.0.0
app-a-v1.0.1
app-b-v1.0.0
```

沒有：

```text
app-a-v1.1.0
```

### 原因與限制

Release Please 的 Manifest Mode 主要以 Release PR 作為版本計算與發布流程的狀態轉換。`skip-github-pull-request: true` 雖然文件描述為跳過 Release PR，但在多專案 Manifest Mode 中可能只完成版本分析，不完成直接 Tag/Release。

官方 Action Issue #906 記錄了相同問題：設定 `skip-github-pull-request: true` 後 workflow 成功，但沒有建立 Tag 或 Release，該問題目前已關閉且沒有規劃修正：[skip-github-pull-request not adding tag or creating release](https://github.com/googleapis/release-please-action/issues/906)

### 目前結論

`skip-github-pull-request: true` 不適合用來實作此 POC 的多專案直接發布流程。

最穩定的 Release Please 流程是：

```text
功能 PR 合併 main
→ Release Please 建立 Release PR
→ 合併 Release PR
→ 建立 component Tag 與 GitHub Release
```

## 問題五：Node.js 20 deprecation warning

### 現象

Actions 顯示：

```text
Node 20 is deprecated.
```

### 影響

目前只是 warning，workflow 仍可成功執行。這是 `googleapis/release-please-action@v4` 內部使用 Node.js 20 造成的 runner 相容性提示，並非本 POC 版本計算失敗的原因。

## 建議方案

### 方案 A：繼續使用 Release Please

移除或設定：

```yaml
skip-github-pull-request: false
```

保留：

```yaml
include-component-in-tag: true
```

這是目前最穩定、最符合 Release Please Manifest Mode 設計的方案。

### 方案 B：完全省略 Release PR

需要改用支援獨立發布的工具，例如 Nx Release，或自行撰寫 workflow。自訂流程需要自行處理：

1. 找出變更的 App
2. 讀取該 App 的上一個 Tag
3. 解析 Conventional Commit
4. 計算 SemVer
5. 建立獨立 Tag 與 GitHub Release
6. 產生版本證據 Artifact

Nx 官方支援 Independent Release，並會為各專案建立獨立 Git Tag：[Release Projects Independently](https://nx.dev/docs/guides/nx-release/release-projects-independently)
