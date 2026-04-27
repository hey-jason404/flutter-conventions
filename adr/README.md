# Architecture Decision Records

本目錄收錄 flutter-conventions 所有設計決策的 rationale。每份 ADR 用 [MADR](https://adr.github.io/madr/) 標準格式。

---

## 索引

| # | Title | Status | Topic |
|---|-------|--------|-------|
| [001](./001-bloc-not-cubit.md) | BLoC over Cubit | Accepted | state-management |
| [002](./002-bloc-event-plain-sealed-class.md) | BLoC Event: Plain Sealed Class | Accepted | state-management |
| [003](./003-bloc-state-freezed-sealed-class.md) | BLoC State: Freezed Sealed Class | Accepted | state-management |
| [004](./004-bloc-event-past-tense-naming.md) | BLoC Event Past Tense Naming | Accepted | state-management |
| [005](./005-imperative-actions-as-side-effect.md) | Imperative Actions as SideEffect | Accepted | state-management |
| [006](./006-side-effect-plain-sealed-class.md) | SideEffect: Plain Sealed Class | Accepted | state-management |
| [007](./007-either-pattern-for-domain-results.md) | Either Pattern for Domain Results | Accepted | error-handling |
| [008](./008-named-parameters-everywhere.md) | Named Parameters Everywhere | Accepted | coding-style |
| [009](./009-usecase-call-method-with-input.md) | UseCase: `call()` Method with Input Class | Accepted | domain-layer |
| [010](./010-no-build-prefix-private-method.md) | No `_build` Prefix Private Methods for Widgets | Accepted | presentation-layer |
| [011](./011-domain-zero-flutter-imports.md) | Domain Layer: Zero Flutter Imports | Accepted | architecture |
| [012](./012-features-no-cross-import.md) | Features Never Cross-Import | Accepted | architecture |

---

## 加新 ADR 流程

1. 確認觸發條件成立（規則背後有踩坑歷史 / 有實質 alternatives 比較 / 是爭議性選擇 / 是團隊立場）。純格式規範不開 ADR。
2. 取下一個流水號（看本檔案最後一筆）
3. 從上方任一 ADR 複製檔案做模板，調整內容
4. 必須有：Status、Context、Decision、Consequences、Alternatives Considered、Related
5. 更新本索引（追加一行）
6. 在對應規範檔加 → ADR 連結
7. PR review，merge

---

## ADR 觸發條件

開 ADR 的條件（任一即可）：

- 規則背後有**踩坑歷史**或具體事件
- 有實質的 **alternatives 比較**值得記錄
- 是**爭議性 / 反直覺**的選擇（讀者第一反應可能是「為什麼這樣？」）
- 規則屬於**團隊立場**（如 Bloc / fpdart 等選型決策）

不開 ADR：純格式 / 業界慣例規範（如 `snake_case` 檔名、`PascalCase` class 名、import 排序）。

---

## ADR 編號規則

- **流水號**：3 位數（`001`、`002`、...）
- **不可重用**：被 supersede 的 ADR 保留檔案，新增的 ADR 編號繼續往後
- 被 supersede 的 ADR `Status` 改為 `Superseded by ADR-XXX`，但檔案保留
