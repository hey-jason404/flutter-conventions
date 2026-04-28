# Flutter Conventions

> 一份為 AI 開發代理（Claude Code、Codex、Cursor）設計的 Flutter / Dart **Mobile App** 開發規範（iOS / Android）。也適用於人類工程師閱讀。
>
> ⚠️ 不適用於 Flutter Web / Desktop。

[![License](https://img.shields.io/github/license/hey-jason404/flutter-conventions)](LICENSE)

---

## 這是什麼

這個 repo 收錄一套**完整、有立場（opinionated）**的 Flutter / Dart 工程規範，**範圍限定 iOS / Android Mobile App**。

設計目標：

- ✅ **單一來源（SSOT）** —— 多個 Flutter repo 透過 git submodule 共用同一份規範
- ✅ **AI 友善** —— 結構化、可被 Claude / Codex / Cursor 直接消費
- ✅ **人類友善** —— 繁中為主，含完整 ✅/❌ 範例 + 為什麼這樣選的 ADR

---

## 立場（必讀）

這份規範**有強烈的工具與模式立場**。如果你的團隊用不同的選擇（例如 Riverpod、GetX、Provider），這份規範**不適合直接套用**。

| 主題 | 我們的選擇 |
|---|---|
| 狀態管理 | `flutter_bloc`（不用 Cubit、不用 Riverpod） |
| 函數式錯誤處理 | `fpdart`（`Either<AppFailure, T>`） |
| DI | `get_it` + `injectable` |
| 網路 | `dio` |
| 導航 | `go_router`（業務導航走 SideEffect） |
| Codegen | `freezed` / `json_serializable` / `slang` / `flutter_gen` |
| 測試 | `bloc_test` + `mocktail` |

完整套件白名單見 [`packages.md`](./packages.md)，每項選擇的 rationale 見 [`adr/`](./adr/)。

---

## 規範地圖

```text
architecture/        架構規範：分層、依賴、檔案結構、DI
patterns/            設計模式：BLoC、錯誤處理、導航、codegen、testing
infrastructure/      基礎設施：HTTP、session、config、platform
style/               程式碼風格：命名、排版
packages.md          套件白名單
adr/                 架構決策紀錄（為什麼這樣選）
templates/           給下游 repo 的入口檔模板
conventions.md       主索引（被下游 @-import）
```

---

## 4. 如何使用

把本 repo 用 git submodule 接到你的 Flutter repo，依下列步驟操作。

### 4.1 加 submodule

在你的 Flutter repo 根目錄執行：

```bash
git submodule add https://github.com/hey-jason404/flutter-conventions.git shared/flutter-conventions
```

> ⚠️ **一定用 HTTPS，不要用 SSH**。
> flutter-conventions 是 public repo，HTTPS 不需要任何 GitHub auth；SSH 反而要求每位團隊成員都先設好 GitHub SSH key，徒增摩擦。

#### Path 選擇

`shared/flutter-conventions/` 是建議的預設 path。依你 repo 結構可選別的：

| 你的 repo 結構 | 建議 path |
|---|---|
| 標準 Flutter（root 直接 `lib/`、無 wrapper） | `shared/flutter-conventions/`（預設） |
| 有 `src/` 包住 Flutter source（如 monorepo） | `src/docs/conventions/` |
| 有 root 級 `docs/` 已存在 | `docs/conventions/` |

選定後請記得在後續模板（§5）中改成你實際的 path。

### 4.2 團隊 onboard

#### 首次 clone repo

```bash
git clone --recurse-submodules <your-repo-url>
```

`--recurse-submodules` 會一併拉 conventions 內容。

#### 已 clone 過 / 切到含 submodule 的 branch

```bash
git submodule update --init --recursive
```

#### 一勞永逸（建議全域設一次）

```bash
git config --global submodule.recurse true
```

之後 `git pull` / `git checkout` / `git switch` 會自動同步 submodule。整個團隊每人設一次，永遠不再踩「忘了 init」這個坑。

### 4.3 更新同步

flutter-conventions 改了之後，下游 repo 二選一：

#### A. Pin 版本（推薦）

每次手動 bump，commit 走 review：

```bash
cd shared/flutter-conventions
git pull origin master
cd ../..
git add shared/flutter-conventions
git commit -m "chore: bump flutter-conventions"
```

✅ 規範變更走 PR review、版本可追蹤、不會被上游 breaking change 突襲。

#### B. 跟最新

```bash
git submodule update --remote --merge
```

✅ 永遠最新；❌ 上游壞了你跟著壞。

> **預設用 A**（pin 版本）。團隊成熟後可考慮 B。

### 4.4 移除

不再使用本規範時，依下列步驟完整移除 submodule：

```bash
# 1. 取消 submodule 註冊
git submodule deinit -f shared/flutter-conventions

# 2. 從 working tree 與 .gitmodules 移除
git rm -f shared/flutter-conventions

# 3. 清理 .git/modules 殘留
rm -rf .git/modules/shared/flutter-conventions

# 4. Commit
git commit -m "chore: remove flutter-conventions"
```

> 📌 若你依 §5 設了 `CLAUDE.md` / `AGENTS.md`，需一併刪除這兩個檔案，或編輯掉裡面引用 `shared/flutter-conventions/` 的段落。

---

## 5. 進階：讓 AI 主動套用規範（建議）

§4 完成後，規範本體已經接到你的 repo。但要讓 Claude Code / Codex / Cursor **每次寫 code 都自動讀規範**，建議再多一步：複製本 repo 的入口模板到你的 Flutter repo root。

> 💡 此步驟非必要。不做也能讓工程師手動參閱規範；做了之後 AI 編輯 `.dart` 檔時會自動套用。

### 適用工具對照

| 你的 AI 工具 | 要建的入口檔 |
|---|---|
| Claude Code | `CLAUDE.md` |
| Codex CLI、Cursor | `AGENTS.md` |
| 多種工具混用 | 兩個都建 |

### 步驟

```bash
# 從 submodule 複製模板到 repo root（兩個都複製或選一個）
cp shared/flutter-conventions/templates/CLAUDE.md.template ./CLAUDE.md
cp shared/flutter-conventions/templates/AGENTS.md.template ./AGENTS.md
```

模板來源連結：
- [`templates/CLAUDE.md.template`](./templates/CLAUDE.md.template)
- [`templates/AGENTS.md.template`](./templates/AGENTS.md.template)

打開複製出來的 `CLAUDE.md` / `AGENTS.md`，做三件事：

1. 把 `{{Project Name}}` 換成你的專案名
2. **若 mount 在非預設 path**（例如 `src/docs/conventions/`），把模板內所有 `shared/flutter-conventions/` 改成你實際的 path
3. 填寫 `Project Context` 段（這個 repo 的背景資訊，不是規範）

最後 commit：

```bash
git add CLAUDE.md AGENTS.md
git commit -m "chore: add AI agent entry files"
```

---

## 問題回報

有任何使用上的疑問、bug、或想討論規範內容，請到 [GitHub Issues](https://github.com/hey-jason404/flutter-conventions/issues) 提報。

---

## 授權

[MIT](./LICENSE)
