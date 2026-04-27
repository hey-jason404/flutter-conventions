# 整合教學：把 flutter-conventions 加進你的 Flutter repo

本文件說明如何把 `flutter-conventions` 整合進一個 Flutter 專案 repo，讓 Claude Code / Codex / Cursor 等 AI 代理在編輯 Dart code 時自動套用團隊規範。

---

## 整合步驟

### 1. 加 submodule

```bash
cd your-flutter-repo
git submodule add https://github.com/hey-jason404/flutter-conventions shared/flutter-conventions
```

> 預設建議路徑：`shared/flutter-conventions/`。
> 你可以選別的路徑，例如 `src/docs/conventions/` —— 只要記得更新模板裡的 path。

### 2. 複製模板

依你和團隊使用的 AI 工具選擇要建哪幾份入口檔：

| 工具 | 入口檔 |
|---|---|
| Claude Code | `CLAUDE.md` |
| Codex CLI、Cursor | `AGENTS.md` |
| 多種工具混用 | 兩個都建 |

```bash
cp shared/flutter-conventions/templates/CLAUDE.md.template CLAUDE.md
cp shared/flutter-conventions/templates/AGENTS.md.template AGENTS.md
```

### 3. 編輯模板

打開 `CLAUDE.md` / `AGENTS.md`，做兩件事：

- 把 `{{Project Name}}` 換成你的專案名
- （如果 submodule 路徑非預設）更新 import / 引用 path
- 填寫 `Project Context`（你的 repo 背景，不是規範）

### 4. Commit 整合結果

```bash
git add .gitmodules shared/flutter-conventions CLAUDE.md AGENTS.md
git commit -m "chore: adopt flutter-conventions"
```

---

## 全域 Git 設定建議

在你的開發機跑一次（之後所有 submodule repo 都受惠）：

```bash
git config --global submodule.recurse true
```

這會讓 `git pull` / `git checkout` 自動同步 submodule，新人少踩一個坑。

---

## CI 配置

### GitHub Actions

```yaml
- uses: actions/checkout@v4
  with:
    submodules: recursive
```

### GitLab CI

```yaml
variables:
  GIT_SUBMODULE_STRATEGY: recursive
```

### Bitbucket Pipelines

```yaml
clone:
  depth: full
options:
  submodules:
    update: true
    strategy: recursive
```

---

## 常見坑

### Clone 後規範資料夾是空的

```bash
git submodule update --init --recursive
```

或者下次直接：

```bash
git clone --recurse-submodules <repo>
```

### 規範改了，本地 repo 沒更新

兩種更新策略二選一：

**Pin 版本（穩定，需手動 bump）**

```bash
cd shared/flutter-conventions
git pull origin master
cd ../..
git add shared/flutter-conventions
git commit -m "chore: bump flutter-conventions"
```

**跟最新（活，每次拉最新）**

```bash
git submodule update --remote --merge
```

---

## 確認規範生效

打開你的 AI 工具，問一個 Flutter 相關問題（例如「幫我建一個 Login Page」）。如果規範生效，AI 應該會：

- 用 `flutter_bloc`（不用 Cubit）
- 用 `Either<AppFailure, T>`
- 不用 `_buildXxx()` 拆 widget
- 命名遵守團隊規範

如果 AI 行為跟規範不一致，檢查：

1. `CLAUDE.md` / `AGENTS.md` 的 import / 引用 path 是否正確
2. AI 工具是否真的讀到該檔（Claude Code 可用 `/memory` 確認）
3. submodule 是否確實 init 了（`ls shared/flutter-conventions/` 應該看到內容）
