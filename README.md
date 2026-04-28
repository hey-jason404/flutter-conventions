# Flutter Conventions

> 一份為 AI 開發代理（Claude Code、Codex、Cursor）設計的 Flutter / Dart 開發規範。也適用於人類工程師閱讀。

[![License](https://img.shields.io/github/license/hey-jason404/flutter-conventions)](LICENSE)

---

## 這是什麼

這個 repo 收錄一套**完整、有立場（opinionated）**的 Flutter / Dart 工程規範。

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

```
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

## 如何使用

### 給工程師：直接讀

從 [`conventions.md`](./conventions.md) 開始，那是主索引。

### 給團隊：透過 git submodule 共用

```bash
# 在你的 Flutter repo 裡（用 HTTPS，不需 GitHub auth）
git submodule add https://github.com/hey-jason404/flutter-conventions.git src/docs/conventions

# 複製入口模板（依使用工具選一或全選）
cp src/docs/conventions/templates/CLAUDE.md.template ./CLAUDE.md
cp src/docs/conventions/templates/AGENTS.md.template ./AGENTS.md

# 編輯 ./CLAUDE.md / ./AGENTS.md，填上專案名與 Project Context
```

> 💡 用 HTTPS 即可，**不需要任何 GitHub 帳號 / SSH key**（本 repo 為 public）。SSH 反而會逼每個團隊成員都先設好 GitHub key，徒增摩擦。

詳細整合說明（含 onboard / CI / 常見坑）見 [`templates/README.md`](./templates/README.md)。

### 給 AI agents：透過模板自動載入

下游 repo 設好後，Claude Code / Codex / Cursor 編輯 `.dart` 檔時會自動套用本規範。

---

## 規範變更流程

1. 開 issue 描述要改什麼、為什麼
2. PR 修改規範內容 + 對應 ADR（如為重要決策）
3. Review → merge
4. 各下游 repo 同步新版（兩種策略二選一，**預設推薦 A**）：
   - **A. Pin 版本**（穩定、走 PR review）：`cd <submodule> && git pull origin master && cd .. && git add <submodule> && git commit -m "chore: bump flutter-conventions"`
   - **B. 跟最新**（活，自動拉最新）：`git submodule update --remote --merge`

---

## 授權

[MIT](./LICENSE)
