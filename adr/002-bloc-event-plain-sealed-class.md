# ADR-002: BLoC Event Plain Sealed Class

## Status

Accepted | Date: 2024-04-10 | Supersedes: -

## Context

BLoC Event 是「使用者剛做了什麼」的訊息物件。物件特性：

- 一次性發出、由 BLoC handler 消費後即可丟棄
- 不需要 `copyWith`（沒有「漸進更新」的場景）
- 不需要 value-equality（事件本身不會被比對 prev/curr）
- 不需要 JSON serialization（不持久化、不跨進程傳遞）

`@freezed` 提供的核心價值是：

- 自動生成 `==`、`hashCode`、`copyWith`、`toString`
- 可選擇生成 `fromJson` / `toJson`

**Event 不需要這些**。給 Event 加 `@freezed` 等於：

- 多一份 codegen 等待時間（每次 Event 變更要跑 build_runner）
- 多一份 `_$Event.freezed.dart` 檔案
- 多一個 `with _$Event` mixin pattern 噪音

## Decision

**BLoC Event 一律使用 plain sealed class，禁止使用 `@freezed`。**

```dart
// ✅
sealed class LoginEvent {}

final class LoginSubmitted extends LoginEvent {
  const LoginSubmitted({required this.email, required this.password});
  final String email;
  final String password;
}
```

```dart
// ❌
@freezed
sealed class LoginEvent with _$LoginEvent {
  const factory LoginEvent.submitted({...}) = LoginSubmitted;
}
```

## Consequences

### Positive

- 啟動 / build 速度更快（少一份 codegen）
- 檔案數量減少
- 寫法直觀，新人可立即理解

### Negative / Trade-offs

- 失去自動 `==`（影響：BLoC 內部 dedup 可能不準）
  - 緩解：dedup 應從 UI 層解決（debounce、disable button），而不是依賴 Event equality

## Alternatives Considered

### `@freezed` on Event

**為什麼不選**：詳見 Context。codegen overhead 換不到 Event 真實需要的價值。

### 普通 class（非 sealed）

**為什麼不選**：失去 sealed class 的 exhaustiveness check。BLoC handler 用 `switch (event)` 配 sealed class 才能在編譯期捕捉漏處理的 Event。

## Related

- 規範：[`patterns/state-management.md`](../patterns/state-management.md)
- 相關 ADR：
  - [ADR-001: BLoC over Cubit](./001-bloc-not-cubit.md)
  - [ADR-003: BLoC State Freezed Sealed Class](./003-bloc-state-freezed-sealed-class.md)（State 為什麼不同對待）
  - [ADR-006: SideEffect Plain Sealed Class](./006-side-effect-plain-sealed-class.md)（SideEffect 同理）
