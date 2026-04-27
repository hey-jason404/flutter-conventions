# ADR-003: BLoC State Freezed Sealed Class

## Status

Accepted | Date: 2024-04-10 | Supersedes: -

## Context

BLoC State 跟 Event 不一樣（→ [ADR-002](./002-bloc-event-plain-sealed-class.md)）。State 物件特性：

- **持久存在**（直到下一個 State 取代）
- 需要 `copyWith`（中間狀態漸進更新，譬如「已 loaded、正在加購物車」這種 sub-state 變化）
- 需要 value-equality（`BlocBuilder` 用 `==` 比對 prev / curr State 決定是否 rebuild）
- 需要正確的 `hashCode`（用於 collection 比對）

手刻這些容易出錯：

- 加新 field 漏改 `==` / `hashCode`
- `copyWith` 寫到後期非常 boilerplate
- 漏改 `toString` 影響 debug

`@freezed` 自動產生這些，且**不會漏**新 field。

## Decision

**BLoC State 一律使用 `@freezed sealed class`，禁止使用 plain sealed class。**

```dart
// ✅
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState.initial() = LoginInitial;
  const factory LoginState.loading() = LoginLoading;
  const factory LoginState.success({required User user}) = LoginSuccess;
  const factory LoginState.failure({required AppFailure failure}) = LoginFailure;
}
```

```dart
// ❌
sealed class LoginState {}
final class LoginSuccess extends LoginState {
  const LoginSuccess({required this.user});
  final User user;
  // 還要手刻 ==、hashCode、copyWith、toString
}
```

## Consequences

### Positive

- `copyWith` 自動生成（new field 自動納入）
- `==` / `hashCode` 自動正確 → `BlocBuilder` 比對最佳化
- `toString` 提供良好 debug 訊息
- 加新 field 不會漏改

### Negative / Trade-offs

- 多一層 codegen 等待時間
- 每次 State 變更都要跑 `build_runner`
- 需要 `with _$LoginState` mixin pattern（語法噪音）

## Alternatives Considered

### Plain sealed class（手刻 `==` / `hashCode` / `copyWith`）

**為什麼不選**：

- 手刻容易遺漏新 field（最常見：加 field 忘記改 `==`，rebuild 不觸發）
- `copyWith` boilerplate 滿屏
- 多 State variant 時各刻一份累

### `equatable` 套件 + 手刻 `copyWith`

**為什麼不選**：

- `equatable` 解決 `==` / `hashCode`，但 `copyWith` 還是要手刻
- 一份 codegen + 一份手刻 = 兩種風格混搭
- freezed 一個套件涵蓋所有需求，更乾淨

## Related

- 規範：[`patterns/state-management.md`](../patterns/state-management.md)、[`patterns/code-generation.md`](../patterns/code-generation.md)
- 對比 ADR：[ADR-002: BLoC Event Plain Sealed Class](./002-bloc-event-plain-sealed-class.md)（Event 為何不用 freezed）
