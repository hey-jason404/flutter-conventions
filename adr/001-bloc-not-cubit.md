# ADR-001: BLoC over Cubit

## Status

Accepted | Date: 2024-03-15 | Supersedes: -

## Context

`flutter_bloc` 套件提供兩種狀態管理工具：

- `Cubit<State>` —— 方法直接觸發 state 變更（簡化版）
- `Bloc<Event, State>` —— 輸入 Event、輸出 State（完整版）

我們團隊在實踐中觀察到使用 Cubit 出現的問題：

1. **意圖記錄遺失**：Cubit 寫到後期常因「需要紀錄使用者意圖」而退化成「在 Cubit 方法裡開 switch case 判斷意圖」—— 等於把 Event 模式手刻在 Cubit 裡
2. **BDD 對齊失去天然錨點**：QA 寫 acceptance criteria 時，Event 提供 `Given ... When LoginSubmitted ... Then ...` 的自然對齊；Cubit 沒有 Event 概念，BDD 描述要繞一圈
3. **多步驟 flow 失控**：多步驟 OTP / 表單，Cubit 中間狀態管理變雜
4. **debug 困難**：沒有 Event 序列，邊界 case 重現只能靠口述操作步驟

## Decision

**所有 BLoC 一律使用 `Bloc<Event, State>`，禁止使用 `Cubit`。**

包括：

- 單純 toggle / 單一動作的 BLoC 也用 `Bloc`
- 沒有「太簡單可以省略 Event」的例外

## Consequences

### Positive

- Event 提供天然的 BDD 對齊點
- 操作序列可被 log、replay、用於分析
- 多步驟 flow 自然映射成 Event sequence
- 跨 BLoC 風格統一，新成員學一次走遍

### Negative / Trade-offs

- 簡單情境多寫一個 Event class（成本：~5 行 code）
- 對只接觸過 Cubit 的工程師需要 onboarding
- BLoC 程式碼總量略多

## Alternatives Considered

### Cubit

**為什麼不選**：上述 Context 中描述的退化模式在多次實踐中重複發生。即使約定「簡單情境用 Cubit」，邊界判斷主觀，後期常需重構成 Bloc。

### 兩者並存（簡單情境 Cubit、複雜情境 Bloc）

**為什麼不選**：「簡單」與「複雜」是相對概念。Cubit 開始的專案後期常因需求變化要遷成 Bloc，重構成本 > 一開始就 Bloc 的多寫成本。

### 自定 state machine 套件（如 `flutter_bloc` 之外的其他選擇）

**為什麼不選**：`flutter_bloc` 在社群成熟、文件完善、tooling 支援好（`bloc_test`、DevTools 整合），換套件是淨損失。

## Related

- 規範：[`patterns/state-management.md`](../patterns/state-management.md)
- 後續 ADR：
  - [ADR-002: BLoC Event Plain Sealed Class](./002-bloc-event-plain-sealed-class.md)
  - [ADR-003: BLoC State Freezed Sealed Class](./003-bloc-state-freezed-sealed-class.md)
  - [ADR-005: Imperative Actions as SideEffect](./005-imperative-actions-as-side-effect.md)
