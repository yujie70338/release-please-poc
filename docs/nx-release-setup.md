# Nx Release POC 設定與問題紀錄

本文件記錄此 Monorepo Docker Image POC 使用 Nx Release 的設定、測試方式，
以及實際遇到的問題。App A 和 App B 使用獨立的版本序列與 Git Tag。

## 目前架構

| Project | 路徑 | 版本檔案 | Tag 格式 |
| --- | --- | --- | --- |
| `app-a` | `images/app-a` | `images/app-a/package.json` | `app-a-v{version}` |
| `app-b` | `images/app-b` | `images/app-b/package.json` | `app-b-v{version}` |

目前仍保留 Release Please 設定，供比較兩種方案時參考：

- `release-please-config.json`
- `.release-please-manifest.json`
- `docs/release-please-issues.md`

## 必要設定

### Root package.json

Root `package.json` 必須包含 Nx 和 JavaScript release plugin：

```json
{
  "name": "release-please-poc",
  "private": true,
  "devDependencies": {
    "@nx/js": "^23.2.0",
    "nx": "^23.2.0"
  }
}
```

`@nx/js` 提供 Nx 預設的 JavaScript version actions。沒有它時，Nx 會顯示：

```text
Unable to resolve the default "versionActions" implementation for project
"app-a" at path: "@nx/js/src/release/version-actions"
```

### 每個 service 的 package.json

每個要由 Nx Release 管理的 project 都需要一個包含 `name` 和 `version` 的
`package.json`：

```json
{
  "name": "app-a",
  "version": "1.0.0"
}
```

`project.json` 只描述 Nx project 的名稱與路徑，不能單獨提供 Nx Release 要更新的
版本欄位。缺少 service 的 `package.json` 時，Nx 可能會正確計算新版本，卻顯示：

```text
No files changed as a result of running versioning
```

### nx.json

目前的 `nx.json` 使用 independent release，並將 project 名稱放進 Tag：

```json
{
  "release": {
    "projects": ["app-a", "app-b"],
    "projectsRelationship": "independent",
    "version": {
      "conventionalCommits": true
    },
    "releaseTag": {
      "pattern": "{projectName}-v{version}"
    },
    "changelog": {
      "projectChangelogs": {
        "file": "CHANGELOG.md",
        "createRelease": "github"
      }
    }
  }
}
```

`projectsRelationship: "independent"` 讓 App A 和 App B 分別計算版本。App B 的
發布不會讓 App A 跳號。

## GitHub Actions

`.github/workflows/release.yml` 在 `main` 收到 push 時執行：

```yaml
on:
  push:
    branches:
      - main
```

Workflow 執行：

```bash
npx nx release version --git-commit --git-tag --git-push
```

流程如下：

1. 將功能 PR 合併到 `main`。
2. `push` 事件啟動 GitHub Actions。
3. Nx 解析 Conventional Commits，更新有變更的 service 版本。
4. Nx 建立版本提交、Tag，並推送回 Repository。

Workflow 使用以下權限：

```yaml
permissions:
  contents: write
```

GitHub Actions 只在 CI 中執行 `npm install`；本機不需要安裝依賴才能建立測試
branch。

## 版本規則

當前版本為 `1.0.0` 時：

| Conventional Commit | 版本變更 | 例子 |
| --- | --- | --- |
| `fix: ...` | Patch | `1.0.0` → `1.0.1` |
| `feat: ...` | Minor | `1.0.0` → `1.1.0` |
| `feat!: ...` | Major | `1.0.0` → `2.0.0` |

也可以使用 footer 表示 breaking change：

```text
feat: update app-a

BREAKING CHANGE: change app-a configuration format
```

## 測試命令

以下命令測試 App B 的 Minor release：

```bash
git switch main
git pull --ff-only origin main
git switch -c test/app-b-feat
printf '\nMinor release test for app-b\n' >> images/app-b/README.md
git add images/app-b/README.md
git commit -m "feat: update app-b"
git push -u origin test/app-b-feat
gh pr create --base main --head test/app-b-feat \
  --title "feat: update app-b" \
  --body "Test Nx Release minor version for app-b"
```

合併 PR 後檢查 Workflow、Tag 和 Artifact：

```bash
gh run list --workflow release.yml
gh release list
git fetch --tags
git tag --list 'app-b-v*' --sort=version:refname
```

預期新增 `app-b-v1.1.0`，而 App A 的 Tag 不變。

## 實際遇到的問題

### 1. Release options 放在錯誤的命令層級

曾使用：

```bash
npx nx release --yes --skip-publish --git-commit --git-tag --git-push
```

Nx 回報：

```text
Unknown arguments: gitCommit, gitTag, gitPush
```

`--git-commit`、`--git-tag` 和 `--git-push` 是 `nx release version` 的 options，
不是頂層 `nx release` 的 options。正確命令是：

```bash
npx nx release version --git-commit --git-tag --git-push
```

### 2. 缺少 `@nx/js`

第一次執行時，Nx 無法解析 `app-a` 的預設 `versionActions`。將 `@nx/js` 加入
root `devDependencies` 後，Nx 才能處理 service 的 `package.json` 版本。

### 3. Nx 計算版本但沒有修改檔案

當 project 只有 `project.json` 而沒有 service `package.json` 時，Actions log 顯示：

```text
Resolved current version ... new version 1.1.0
No files changed as a result of running versioning
```

新增每個 service 的 `package.json` 後，Nx 才有實際版本檔案可以更新，並能建立
版本提交與 Tag。

### 4. Tag-only 不符合目前流程

目前命令包含 `--git-commit`，所以 Nx 會建立版本提交。這個提交保存
`package.json` 與 changelog 的版本變更，Tag 會指向該提交。

若移除 `--git-commit`，版本檔案可能留在未提交狀態，而 Tag 會指向舊的 `HEAD`。
因此 POC 保留完整流程：

```text
版本檔案更新 → release commit → Git Tag → push
```

若需求是完全不修改版本檔案、只建立 Tag，則需要自訂腳本計算 SemVer 和建立 Tag；
這不屬於 Nx Release 的標準版本流程。

## 參考資料

- [Release projects independently](https://nx.dev/docs/guides/nx-release/release-projects-independently)
- [Automatically version with Conventional Commits](https://nx.dev/docs/guides/nx-release/automatically-version-with-conventional-commits)
- [Nx commands](https://nx.dev/docs/reference/nx-commands)
- [Publish in CI/CD](https://nx.dev/docs/guides/nx-release/publish-in-ci-cd)
- [Automate GitHub Releases](https://nx.dev/docs/kb/automate-github-releases)
