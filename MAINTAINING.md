# Maintaining flutter-conventions

> 本檔案是 AI agent / 人類維護者更動本 repo 規範時的 SOP。確保風格一致、規範無衝突。

---

## §0 與人類維護者對話原則（重要）

收到 issue 或維護任務後，**不要僅憑標題就動手**。流程：

1. 跑 §1 動工前 check
2. 若任何模糊（scope、優先順序、立場性決策、與既有規則衝突的判斷）→ **停下來**，在 issue 留言或回 prompt 問人類維護者
3. 收到回覆後再動手

> 💡 **規範調整本來就會需要往返討論。** 寧可多問一次，也不要做出跟立場矛盾的改動。

### 搭配 superpowers plugin（optional）

若維護者環境有 `superpowers` plugin，建議流程：

- `brainstorming` —— 處理需求對話，把模糊點問清楚再進入設計
- `writing-plans` —— 產出詳細實作 plan
- `requesting-code-review` —— commit 前自我驗證

沒有 plugin 也能照本 SOP 走，只是改成手動。

---

## §1 動工前 check（pre-flight）

改任何規範前**必做**：

1. 讀對應的既有規範檔（`architecture/` / `patterns/` / `infrastructure/` / `style/` / `packages.md`），確認新需求跟現有規則沒有矛盾
2. 讀 [`adr/`](./adr/) 索引，找有沒有相關 ADR 已涵蓋此主題
3. 若有衝突 → 決定要 supersede 既有 ADR 還是調整新需求；決策需在 issue / commit 留紀錄
4. pre-flight 完成後仍有模糊 → 回 §0 與維護者對話

---

## §2 索引與交叉引用同步義務

> **總原則**：任何檔案 rename / split / merge / delete 時，先 `grep -rn` 整個 repo 找出所有舊路徑 / 舊檔名引用，逐一修正。下表列常見情境。

| 你改了什麼 | 必須同步檢查 / 更新 |
|---|---|
| 加新 ADR | (a) `adr/index.md` 索引表<br>(b) 對應規範檔頂部「規則表 → ADR」加連結<br>(c) 對應規範檔內加 inline `→ 詳見 [ADR-XXX]`（如適用）<br>(d) 其他相關 ADR 的 `Related` section 補新 ADR 引用 |
| Supersede 既有 ADR | (a) 新 ADR 的 Status 標 `Supersedes: ADR-XXX`<br>(b) 舊 ADR 的 Status 改為 `Superseded by ADR-XXX`（**不刪檔、不改其他內容**）<br>(c) `conventions.md` cheatsheet 引用切到新 ADR<br>(d) 規範檔內 inline `→ [舊 ADR]` 切到新 ADR |
| 改 `packages.md` 套件選擇 | (a) 補 / 改對應 ADR<br>(b) root `README.md` 「立場（必讀）」段（如改的是頂層套件）<br>(c) 受影響的規範檔（`grep -rn` 該套件名找全引用） |
| 加 / 拆 / 刪 / 改名 規範檔（`architecture/` / `patterns/` / `infrastructure/` / `style/`） | (a) `conventions.md` 「你要做的事 → 必讀」表的檔名<br>(b) 其他規範檔內的相對連結（`grep -rn './<old>'`）<br>(c) ADR 的 `Related` section 引用 |
| 改 `conventions.md` 「不可妥協」cheatsheet | (a) 對應 ADR 仍 Accepted<br>(b) 對應規範檔仍有該規則 |
| 改 root `README.md` 「立場（必讀）」段 | (a) `packages.md` 一致<br>(b) `conventions.md` 立場聲明段一致 |
| 改 AI 入口設計（mount path 慣例 / `conventions.md` 載入方式 / 模板與 AI 工具互動模式） | (a) `templates/CLAUDE.md.template`（含 `@` 引用語法、預設 path）<br>(b) `templates/AGENTS.md.template`（含必讀提示、預設 path）<br>(c) root `README.md` §4「如何使用」/ §5「進階」步驟是否仍正確 |
| 改 `conventions.md` 檔名 / 路徑 | 🚨 **絕對禁止**。此檔是下游 submodule `@` 引用的固定目標，改名 = break 所有下游 repo。沒有「協調後可改」這個選項。 |

---

## §3 風格 invariants

### 文件本體

- **語言**：繁中為主；技術名詞、套件名、code 用英文
- **Callouts**：用 blockquote 開頭加語義 emoji。不在純段落用「粗體開頭加 emoji」假裝 callout
  - `> ⚠️` —— 警告 / 注意（一般風險或需留意處）
  - `> 📌` —— 補充說明（pin 住的注釋）
  - `> 💡` —— 提示 / 建議
  - `> **🔒 ...**` —— 不可變更 / append-only 原則
  - `> **🚨 ...**` 或表格內 `🚨 ...` —— 絕對禁止 / hard prohibition
- **正反例**：每條規則須有 `✅` / `❌` 範例對照
- **目錄樹**：用 ` ```text ` fence，不用裸 ` ``` `

### 命名

- 檔名 `snake_case.md`（除大寫慣例檔：`README.md` / `LICENSE` / `MAINTAINING.md` / `CLAUDE.md` / `AGENTS.md`）
- ADR 檔名 `<NNN>-<kebab-case-title>.md`，例如 `001-bloc-not-cubit.md`

### 連結

- 內部連結用相對路徑 `./xxx.md`，不用絕對 URL

---

## §4 加新 ADR 流程

> **🔒 ADR 不可變更原則**：ADR 是「決策當時的快照」，append-only。內容（Context / Decision / Consequences / Alternatives）**禁止事後修改**。要變更決策 → 開新 ADR + 標既有 ADR `Superseded by ADR-XXX`，舊檔保留。允許的編輯：Status 變更（→ Superseded）、typo 修正、連結修正。

### 觸發條件（任一即可）

- 規則背後有踩坑歷史或具體事件
- 有實質的 alternatives 比較值得記錄
- 是爭議性 / 反直覺的選擇（讀者可能反問「為什麼這樣？」）
- 規則屬於團隊立場（如 Bloc / fpdart 等選型決策）

**不開 ADR**：純格式 / 業界慣例規範（如 `snake_case` 檔名、import 排序）。

### 流程

1. 取下一個流水號（看 [`adr/`](./adr/) 索引最後一筆 + 1）
2. 從既有 ADR 複製檔案做模板（推薦 `adr/001-bloc-not-cubit.md`）
3. 必含 sections：Status、Context、Decision、Consequences、Alternatives Considered、Related
4. 用 [MADR](https://adr.github.io/madr/) 標準格式
5. 更新 `adr/index.md` 索引（追加一行）
6. 在對應規範檔加 → ADR 連結
7. 視情況同步 §2 表中的其他索引

### 編號規則

- 流水號 3 位數（`001`、`002`、...）
- **不可重用**：被 supersede 的 ADR 保留檔案，新增的 ADR 編號繼續往後
- 被 supersede 的 ADR `Status` 改為 `Superseded by ADR-XXX`，但檔案保留
