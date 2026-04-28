# 整合教學：把 flutter-conventions 加進你的 Flutter repo

本文件說明如何把 `flutter-conventions` 整合進一個 Flutter 專案 repo，讓 Claude Code / Codex / Cursor 等 AI 代理在編輯 Dart code 時自動套用團隊規範。

---

## 整合步驟

### 1. 加 submodule（用 HTTPS）

```bash
cd your-flutter-repo
git submodule add https://github.com/hey-jason404/flutter-conventions.git src/docs/conventions
```

> ⚠️ **一定用 HTTPS，不要用 SSH**。
> flutter-conventions 是 public repo，HTTPS 不需要 GitHub auth；SSH 反而要求每位團隊成員都先設好 GitHub SSH key，徒增摩擦。

> 路徑可調整：常見選擇有 `src/docs/conventions/`、`shared/flutter-conventions/`、`docs/conventions/`。選好後**所有模板裡的 path 都要對應修改**。

### 2. 複製模板

依你和團隊使用的 AI 工具選擇要建哪幾份入口檔：

| 工具 | 入口檔 |
|---|---|
| Claude Code | `CLAUDE.md` |
| Codex CLI、Cursor | `AGENTS.md` |
| 多種工具混用 | 兩個都建 |

```bash
cp src/docs/conventions/templates/CLAUDE.md.template CLAUDE.md
cp src/docs/conventions/templates/AGENTS.md.template AGENTS.md
```

### 3. 編輯模板

打開 `CLAUDE.md` / `AGENTS.md`，做三件事：

- 把 `{{Project Name}}` 換成你的專案名
- 把模板內的預設 path（`shared/flutter-conventions/`）改成你實際 mount 的位置（例如 `src/docs/conventions/`）
- 填寫 `Project Context` 段（這個 repo 的背景資訊，不是規範）

### 4. Commit 整合結果

```bash
git add .gitmodules src/docs/conventions CLAUDE.md AGENTS.md
git commit -m "chore: adopt flutter-conventions"
```

---

## 團隊成員 onboard

### 首次 clone repo

```bash
git clone --recurse-submodules <your-repo-url>
```

`--recurse-submodules` 會一併拉 conventions 內容。

### 已 clone 過 / 切到含 submodule 的 branch

```bash
git submodule update --init --recursive
```

### 一勞永逸（建議全域設一次）

```bash
git config --global submodule.recurse true
```

之後 `git pull` / `git checkout` / `git switch` 會自動同步 submodule。整個團隊每人設一次，永遠不再踩「忘了 init」這個坑。

---

## 規範更新時的同步策略

flutter-conventions 改了之後，下游 repo 二選一：

### A. Pin 版本（穩定，推薦）

每次手動 bump，commit 走 review：

```bash
cd src/docs/conventions
git pull origin master
cd ../../..
git add src/docs/conventions
git commit -m "chore: bump flutter-conventions"
```

✅ 規範變更走 PR review、版本可追蹤、不會被上游 breaking change 突襲。

### B. 跟最新（活，看情況）

```bash
git submodule update --remote --merge
```

✅ 永遠最新；❌ 上游壞了，你跟著壞。

> **預設用 A**（pin 版本）。團隊成熟後可考慮 B。

---

## CI 設定（多數情況不用設）

**Flutter 專案的 CI 通常只跑 `flutter analyze` / `flutter test` / build —— 完全不需要讀規範 markdown，因此不必特別處理 submodule。** 維持 GitLab / GitHub Actions / Bitbucket 預設即可。

只在以下少數情境才需要在 CI init submodule：

- CI 跑 markdown lint 或 doc validation
- CI 自動產生文件 site
- 任何 CI 步驟會 read `src/docs/conventions/` 內容

如果你確定需要，再依平台查 docs（GitLab `GIT_SUBMODULE_STRATEGY`、GitHub Actions `submodules: recursive` 等）。**沒事不要加，只是浪費 CI 時間 + 多失敗點**。

---

## 常見坑（實戰整理）

### 1. `git@github.com: Permission denied (publickey)`

submodule URL 被設成 SSH，但你（或團隊成員）沒設 GitHub SSH key。

**解**：把 URL 改 HTTPS（flutter-conventions 是 public，不需任何 GitHub auth）。

```bash
git submodule set-url src/docs/conventions https://github.com/hey-jason404/flutter-conventions.git
git submodule sync
git submodule update --init --recursive
git add .gitmodules
git commit -m "chore: switch flutter-conventions to HTTPS"
```

### 2. Clone / 切 branch 後 `src/docs/conventions/` 是空資料夾

submodule pointer 在但內容沒拉。

**解**：

```bash
git submodule update --init --recursive
```

長期解：上面提到的 `git config --global submodule.recurse true`。

### 3. flutter-conventions 改了、本地沒同步

submodule 預設 pin 在某個 SHA，`git pull` 不會動 submodule 內容。

**解**：依「規範更新時的同步策略」執行。

### 4. AI 工具沒照規範寫

通常代表 submodule 沒 init，`@import` 失效（檔案不存在）。

**檢查**：

```bash
ls src/docs/conventions/conventions.md   # 必須存在
```

不存在 → 跑 `git submodule update --init --recursive`。

### 5. CI build 變慢、偶有失敗

多半是不必要地加了 `GIT_SUBMODULE_STRATEGY: recursive` / `submodules: recursive`（見上節「CI 設定」）。

**解**：移除該設定，回到 CI 平台預設行為。

---

## 確認規範生效

打開你的 AI 工具，問一個 Flutter 相關問題（例如「幫我建一個 Login Page」）。如果規範生效，AI 應該會：

- 用 `flutter_bloc`（**不**用 Cubit）
- 用 `Either<AppFailure, T>`、用 fpdart
- **不**用 `_buildXxx()` private method 拆 widget（抽獨立 class）
- 命名遵守 `LoginBloc` / `LoginUseCase` / `LoginSubmitted` 等 pattern

如果 AI 行為跟規範不一致：

1. `CLAUDE.md` / `AGENTS.md` 的 import / 引用 path 是否正確（路徑必須對應你實際 submodule 位置）
2. AI 工具是否真的讀到該檔（Claude Code 可用 `/memory` 確認）
3. submodule 是否確實 init（`ls src/docs/conventions/conventions.md` 應該看到檔）
