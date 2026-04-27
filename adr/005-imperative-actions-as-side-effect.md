# ADR-005: Imperative Actions as SideEffect

## Status

Accepted | Date: 2024-04-20 | Supersedes: -

## Context

BLoC 模型中有三種訊息：

- **Event**：使用者剛做了什麼（輸入）
- **State**：BLoC 當前在哪（持久狀態）
- **SideEffect**：BLoC 對外發出的指令（一次性事件）

「導航」「show dialog」「show snackbar」是命令式動作，**屬於 SideEffect**，不該放 Event 或 State。但實務上常見以下反 pattern：

### 反 pattern 1：命令式動詞當 Event

```dart
final class LoginNavigateToHome extends LoginEvent {}        // ❌
```

問題：

- 語意混淆：Event 是「使用者剛做了什麼」，「導航到首頁」不是使用者做的事
- 跟 BLoC 的輸入 / 輸出邊界打架（Event = 輸入，導航 = 輸出）
- 寫到後期，BLoC handler 內部會出現 `add(LoginNavigateToHome())` 這種怪 code

### 反 pattern 2：用 `BlocListener` 監聽 state 跳

```dart
BlocListener<LoginBloc, LoginState>(
  listener: (context, state) {
    if (state is LoginSuccess) context.go('/home');         // ❌
  },
)
```

問題：

- State 是「持久狀態」，不是「一次性事件」
- 重新進入 LoginSuccess（譬如 BLoC 重建）會**重複觸發 navigation**
- 邏輯散落 widget 而非集中在 BLoC

### 反 pattern 3：用 state flag 觸發 navigation

```dart
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState({
    @Default(false) bool shouldNavigate,                     // ❌
    User? user,
  }) = _LoginState;
}
```

問題：

- 一次性事件被當持久 state
- 需要再 emit `shouldNavigate: false` 重置（醜）
- rebuild 時可能重複觸發

### SideEffect 解決問題

- 一次性語意明確（事件流，不是 state）
- 集中在 BLoC handler 內 emit（邏輯不散落）
- Page 訂閱 stream 處理（broadcast，重新進入 page 不會重複觸發舊 effect）

## Decision

**所有命令式動作（imperative verb）一律放 SideEffect，禁止放 Event 或用 state flag 觸發。**

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

// BLoC fold 後 emit
result.fold(
  (failure) => emit(LoginFailure(failure: failure)),
  (user) {
    emit(LoginSuccess(user: user));
    emitSideEffect(const LoginNavigateToHome());     // ✅
  },
);
```

## Consequences

### Positive

- Event / State / SideEffect 三者語意分明
- 一次性事件不污染 state
- BLoC handler 邏輯集中、可測試
- Page 用統一 SideEffect 訂閱機制處理 navigation / dialog / snackbar

### Negative / Trade-offs

- 多一個 SideEffect sealed class（設計成本）
- 需要 `BlocSideEffectsMixin` 之類的 plumbing（一次性投資）
- 對只熟悉 vanilla Cubit 的工程師需要學習

## Alternatives Considered

### 命令式 Event

**為什麼不選**：詳見 Context 反 pattern 1。

### `BlocListener` 監聽 state

**為什麼不選**：State 持久語意 vs 一次性事件邊界混淆；rebuild 重複觸發；詳見 Context 反 pattern 2。

### State flag

**為什麼不選**：詳見 Context 反 pattern 3。

### 把 navigation 放 page-level callback

**為什麼不選**：

- Page `onPressed: () { add(Event); context.go(...) }` 在 outcome 還沒回來就跳
- 業務結果（成功 / 失敗）必須來自 BLoC，page callback 跳是時序錯誤

## Related

- 規範：[`patterns/state-management.md`](../patterns/state-management.md)、[`patterns/navigation.md`](../patterns/navigation.md)
- 相關 ADR：
  - [ADR-004: BLoC Event Past Tense Naming](./004-bloc-event-past-tense-naming.md)
  - [ADR-006: SideEffect Plain Sealed Class](./006-side-effect-plain-sealed-class.md)
