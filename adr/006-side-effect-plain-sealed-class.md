# ADR-006: SideEffect Plain Sealed Class

## Status

Accepted | Date: 2024-04-22 | Supersedes: -

## Context

SideEffect 是 BLoC 對外發出的「一次性指令」（→ [ADR-005](./005-imperative-actions-as-side-effect.md)）。設計時要選擇格式：

- **`enum`** —— 簡單、無 codegen
- **plain sealed class** —— 可帶參數、無 codegen
- **`@freezed sealed class`** —— 可帶參數 + auto-generated `==` / `copyWith`

### `enum` 的限制

```dart
enum LoginSideEffect { navigateToHome, showError }
```

問題：**`enum` 不能帶參數**。但 SideEffect 經常需要帶值：

- `LoginShowError({required String message})` —— 錯誤訊息
- `LoginConfirmAccountLock({required DateTime unlockAt})` —— unlock 時間
- `ProductDetailShowAddedToCart({required Product product})` —— 加進去的商品

`enum` 表達不了這些，只能搭配 redundant state 或外部 lookup，反而複雜化。

### `@freezed` 的不必要

SideEffect 的物件特性跟 Event 相同（→ [ADR-002](./002-bloc-event-plain-sealed-class.md)）：

- 一次性發出，被消費後即可丟棄
- 不需要 `copyWith`（沒有「漸進更新 SideEffect」的場景）
- 不需要 value-equality（SideEffect 不被比對）

加 `@freezed` 是純 codegen overhead。

### Plain sealed class 剛好夠

- 可帶參數
- 無 codegen 等待
- 配 sealed exhaustiveness 處理 dispatch

## Decision

**BLoC SideEffect 一律使用 plain sealed class，禁止 `@freezed`、禁止 `enum`。**

```dart
// ✅
sealed class LoginSideEffect {}

final class LoginNavigateToHome extends LoginSideEffect {
  const LoginNavigateToHome();
}

final class LoginShowError extends LoginSideEffect {
  const LoginShowError({required this.message});
  final String message;
}

final class LoginConfirmAccountLock extends LoginSideEffect {
  const LoginConfirmAccountLock({required this.unlockAt});
  final DateTime unlockAt;
}
```

```dart
// ❌ enum 不能帶參數
enum LoginSideEffect { navigateToHome, showError, confirmAccountLock }

// ❌ @freezed 是 overhead
@freezed
sealed class LoginSideEffect with _$LoginSideEffect {
  const factory LoginSideEffect.navigateToHome() = LoginNavigateToHome;
  const factory LoginSideEffect.showError({required String message}) = LoginShowError;
}
```

## Consequences

### Positive

- 可帶參數，覆蓋所有 SideEffect 場景
- 無 codegen 等待
- 配 sealed exhaustiveness 在 page handler `switch` 內保證完整處理

### Negative / Trade-offs

- 失去自動 `==`（SideEffect 不需要 dedup，可接受）

## Alternatives Considered

### `enum`

**為什麼不選**：無法帶參數。`LoginShowError` 需要 message string、`LoginConfirmAccountLock` 需要 unlockAt timestamp，enum 表達不了。

### `@freezed`

**為什麼不選**：codegen overhead 不換來實際價值（SideEffect 不需要 `copyWith` / `==`）。同 [ADR-002](./002-bloc-event-plain-sealed-class.md) 對 Event 的論點。

### 普通 class（非 sealed）

**為什麼不選**：失去 sealed exhaustiveness check。Page handler `switch (effect)` 必須 exhaustive 才能在編譯期捕捉漏處理的 SideEffect 類型。

## Related

- 規範：[`patterns/state-management.md`](../patterns/state-management.md)
- 相關 ADR：
  - [ADR-002: BLoC Event Plain Sealed Class](./002-bloc-event-plain-sealed-class.md)（Event 同理）
  - [ADR-005: Imperative Actions as SideEffect](./005-imperative-actions-as-side-effect.md)
