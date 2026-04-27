# ADR-004: BLoC Event Past Tense Naming

## Status

Accepted | Date: 2024-04-15 | Supersedes: -

## Context

Event 在 BLoC 裡描述的是「**已經發生的事**」（使用者按了按鈕、表單值變了、頁面 mount 了）。命名有三種反 pattern 常見：

1. **命令式動詞**（`LoginLogin`、`LoginNavigateToHome`）
   - 含義變成「BLoC，做這件事」—— 這是命令，不是事件
   - 跟 SideEffect 撞語意（SideEffect 是 BLoC 對外發出的命令）
   - 實踐中常導致 dev 把「導航」「show dialog」當 Event 加，而不是走 SideEffect → 為何錯詳見 [ADR-005](./005-imperative-actions-as-side-effect.md)

2. **加 `Event` suffix**（`LoginSubmittedEvent`、`LoginEmailChangedEvent`）
   - 父類 sealed class 已經叫 `LoginEvent`，子類重複後綴是冗詞
   - `class LoginSubmittedEvent extends LoginEvent` 看起來像繞口令
   - 業界其他 sealed class 不會這樣寫（`Result.successResult` 不會出現）

3. **不一致時態**（同一個 BLoC 內混合過去式和現在式）
   - `LoginSubmitted` + `LoginSubmit` + `LoginEmail`
   - 讀者要每次判斷「這是 event 還是 something else」

過去式 + 無 suffix 的 pattern：

- 強調「事件」的時間語意（已發生，而非要做）
- 命名簡潔（`LoginSubmitted` < `LoginSubmittedEvent`）
- 跟 BDD `Given...When...Then` 對齊（"When LoginSubmitted"）

## Decision

**BLoC Event 命名遵守 `{Page}{Context?}{PastVerb}` pattern：**

- 一律使用**過去式動詞**
- **禁止** `Event` suffix
- **禁止**命令式動詞當 Event（屬於 SideEffect → ADR-005）

```dart
// ✅
LoginSubmitted
LoginEmailChanged
LoginPasswordChanged
ProfileUpdateRequested
ProductDetailViewed
```

```dart
// ❌ 命令式
LoginLogin
LoginNavigateToHome

// ❌ Event suffix
LoginSubmittedEvent

// ❌ 現在式
LoginSubmit
LoginChangeEmail
```

## Consequences

### Positive

- Event vs SideEffect 邊界清楚（過去式 vs 命令式）
- 命名一致，不需每次判斷時態
- BDD scenario 描述對齊：`When LoginSubmitted ... Then ...`
- 減少冗詞（`Event` suffix）

### Negative / Trade-offs

- 偶爾過去式英文不直觀
  - 例：`LoginEmailChanged` 對 form field 變更稍微 verbose
  - 短形是可接受的權衡，避免每次都 `WasChangedOnLoginPage` 這種長句
- 對 ESL 工程師需要短時間適應

## Alternatives Considered

### 命令式動詞當 Event

**為什麼不選**：詳見 Context #1 + [ADR-005](./005-imperative-actions-as-side-effect.md)。

### 加 `Event` suffix

**為什麼不選**：冗詞，sealed 父類已經提供 type safety；`class LoginSubmittedEvent extends LoginEvent` 的命名重複是純噪音。

### 不規定命名（讓團隊自由發揮）

**為什麼不選**：不同 BLoC 命名不一致，新人讀 code 要每次切換 mental model；命名規範化的成本低、收益高。

## Related

- 規範：[`patterns/state-management.md`](../patterns/state-management.md)、[`style/naming.md`](../style/naming.md)
- 相關 ADR：
  - [ADR-001: BLoC over Cubit](./001-bloc-not-cubit.md)
  - [ADR-005: Imperative Actions as SideEffect](./005-imperative-actions-as-side-effect.md)
