# State Management

> **TL;DR**：一律 `Bloc<Event, State>` + SideEffect 模式。Event 是過去式 plain sealed class，State 是 `@freezed sealed class`，SideEffect 是 plain sealed class（禁 `@freezed`、禁 `enum`）。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | 一律 `Bloc<Event, State>`，禁用 `Cubit` | [ADR-001](../adr/001-bloc-not-cubit.md) |
| 2 | Event 用 plain sealed class，禁 `@freezed` | [ADR-002](../adr/002-bloc-event-plain-sealed-class.md) |
| 3 | State 用 `@freezed sealed class`，禁 plain sealed class | [ADR-003](../adr/003-bloc-state-freezed-sealed-class.md) |
| 4 | Event 命名過去式，禁 `Event` suffix | [ADR-004](../adr/004-bloc-event-past-tense-naming.md) |
| 5 | 命令式動詞屬於 SideEffect 而非 Event | [ADR-005](../adr/005-imperative-actions-as-side-effect.md) |
| 6 | SideEffect 用 plain sealed class，禁 `@freezed` 與 `enum` | [ADR-006](../adr/006-side-effect-plain-sealed-class.md) |
| 7 | BLoC 只依賴 domain UseCase，不依賴 Repository / DataSource | — |

---

## 三件套：Event、State、SideEffect

| | Event | State | SideEffect |
|---|---|---|---|
| 語意 | 使用者剛做了什麼 | BLoC 當前在哪 | BLoC 對外發出的指令 |
| 時態 | 過去式 | 名詞 | 命令式 |
| 格式 | plain sealed class | `@freezed sealed class` | plain sealed class |
| 例 | `LoginSubmitted` | `LoginLoading` | `LoginNavigateToHome` |

---

## Event

### 規則

- **格式**：plain sealed class（**禁** `@freezed`）
- **命名**：`{Page}{Context?}{PastVerb}` —— 過去式
- **目的**：描述「使用者剛做了什麼」（事件，過去發生）
- **禁** `Event` suffix on 變體

### ✅ 正確範例

```dart
sealed class LoginEvent {}

final class LoginSubmitted extends LoginEvent {
  const LoginSubmitted({required this.email, required this.password});
  final String email;
  final String password;
}

final class LoginEmailChanged extends LoginEvent {
  const LoginEmailChanged({required this.email});
  final String email;
}

final class LoginPasswordChanged extends LoginEvent {
  const LoginPasswordChanged({required this.password});
  final String password;
}
```

### ❌ Event 用 `@freezed`

```dart
@freezed
sealed class LoginEvent with _$LoginEvent {
  const factory LoginEvent.submitted({
    required String email,
    required String password,
  }) = LoginSubmitted;
}
```

**為什麼錯**：Event 是輕量訊息物件，不需要 `copyWith` / value-equality；freezed 是純 codegen overhead。
詳見 [ADR-002](../adr/002-bloc-event-plain-sealed-class.md)。

### ❌ 加 `Event` suffix

```dart
final class LoginSubmittedEvent extends LoginEvent {}
final class LoginEmailChangedEvent extends LoginEvent {}
```

**為什麼錯**：父類已經叫 `LoginEvent`，子類重複後綴是冗詞。
詳見 [ADR-004](../adr/004-bloc-event-past-tense-naming.md)。

### ❌ 命令式動詞當 Event

```dart
final class LoginLogin extends LoginEvent {}                     // ❌
final class LoginNavigateToHome extends LoginEvent {}            // ❌
final class LoginShowBiometricDialog extends LoginEvent {}       // ❌
```

**為什麼錯**：「執行登入」、「導航到首頁」、「show dialog」是命令、是輸出指令，不是「使用者剛做了什麼」這個輸入事件。**這些屬於 SideEffect**。
詳見 [ADR-005](../adr/005-imperative-actions-as-side-effect.md)。

---

## State

### 規則

- **格式**：`@freezed sealed class`（**禁** plain sealed class）
- **命名**：`{Page}Initial / Loading / Success / Failure`
- **目的**：描述「BLoC 當前在哪個位置」

### ✅ 正確範例

```dart
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState.initial() = LoginInitial;
  const factory LoginState.loading() = LoginLoading;
  const factory LoginState.success({required User user}) = LoginSuccess;
  const factory LoginState.failure({required AppFailure failure}) = LoginFailure;
}
```

### ❌ State 用 plain sealed class

```dart
sealed class LoginState {}

final class LoginInitial extends LoginState {
  const LoginInitial();
}

final class LoginSuccess extends LoginState {
  const LoginSuccess({required this.user});
  final User user;

  @override
  bool operator ==(Object other) =>
      other is LoginSuccess && other.user == user;

  @override
  int get hashCode => user.hashCode;
}
```

**為什麼錯**：

1. State 需要 **`copyWith`**（中間狀態漸進更新）
2. State 需要 **value-equality**（`BlocBuilder` 比對 prev / curr 決定要不要 rebuild）
3. plain sealed class 手刻 `==` / `hashCode` 容易遺漏新 field、漏寫
4. `@freezed sealed class` 自動正確產生這些

詳見 [ADR-003](../adr/003-bloc-state-freezed-sealed-class.md)。

### State 包含中間資料的場景

如果有「載入中但已知道部分資料」的場景：

```dart
@freezed
sealed class ProductDetailState with _$ProductDetailState {
  const factory ProductDetailState.initial() = ProductDetailInitial;
  const factory ProductDetailState.loading() = ProductDetailLoading;
  const factory ProductDetailState.loaded({
    required Product product,
    required bool isAddingToCart,           // 「加購物車中」是 sub-state
  }) = ProductDetailLoaded;
  const factory ProductDetailState.failure({
    required AppFailure failure,
  }) = ProductDetailFailure;
}

// usage in handler
emit(state.copyWith(isAddingToCart: true));    // freezed copyWith 自動處理
```

---

## SideEffect

### 規則

- **格式**：plain sealed class（**禁** `@freezed`、**禁** `enum`）
- **命名**：`{Page}{ImperativeVerb}{Object?}` —— 命令式動詞
- **目的**：BLoC 對外發出的「一次性指令」（navigate、show dialog、show snackbar）

### ✅ 正確範例

```dart
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

### ❌ SideEffect 用 `@freezed`

```dart
@freezed
sealed class LoginSideEffect with _$LoginSideEffect {
  const factory LoginSideEffect.navigateToHome() = LoginNavigateToHome;
  const factory LoginSideEffect.showError({required String message}) = LoginShowError;
}
```

**為什麼錯**：SideEffect 是輕量訊息，沒 `copyWith` / equality 的需求；codegen overhead 沒必要。

### ❌ SideEffect 用 `enum`

```dart
enum LoginSideEffect { navigateToHome, showError, confirmAccountLock }
```

**為什麼錯**：`enum` **不能帶參數**。`LoginShowError` 需要 message string、`LoginConfirmAccountLock` 需要 unlockAt timestamp，enum 表達不了。
詳見 [ADR-006](../adr/006-side-effect-plain-sealed-class.md)。

---

## BLoC 實作

### 結構

- 一律 `Bloc<Event, State>`，**禁** `Cubit`
- `@injectable`
- 注入只能是 **domain UseCase**（禁 Repository、DataSource）
- SideEffect 透過 mixin / stream 暴露

### ✅ 完整範例

```dart
@injectable
class LoginBloc extends Bloc<LoginEvent, LoginState>
    with BlocSideEffectsMixin<LoginSideEffect> {
  LoginBloc({required LoginUseCase loginUseCase})
      : _loginUseCase = loginUseCase,
        super(const LoginInitial()) {
    on<LoginSubmitted>(_onLoginSubmitted);
    on<LoginEmailChanged>(_onLoginEmailChanged);
    on<LoginPasswordChanged>(_onLoginPasswordChanged);
  }

  final LoginUseCase _loginUseCase;

  Future<void> _onLoginSubmitted(
    LoginSubmitted event,
    Emitter<LoginState> emit,
  ) async {
    emit(const LoginLoading());

    final result = await _loginUseCase(
      input: LoginInput(email: event.email, password: event.password),
    );

    result.fold(
      (failure) {
        emit(LoginFailure(failure: failure));
        emitSideEffect(LoginShowError(message: failure.message));
      },
      (user) {
        emit(LoginSuccess(user: user));
        emitSideEffect(const LoginNavigateToHome());
      },
    );
  }

  void _onLoginEmailChanged(
    LoginEmailChanged event,
    Emitter<LoginState> emit,
  ) {
    // 純 form field 變更不需要 emit state（避免 over-rebuild）
    // 或 emit form-validation state，視 page 設計
  }

  void _onLoginPasswordChanged(...) { ... }
}
```

### `BlocSideEffectsMixin`

提供 SideEffect stream 的 mixin（自家實作或團隊共用）。基本 contract：

```dart
mixin BlocSideEffectsMixin<E> on BlocBase {
  final StreamController<E> _controller = StreamController<E>.broadcast();

  Stream<E> get sideEffects => _controller.stream;

  void emitSideEffect(E effect) {
    if (!_controller.isClosed) _controller.add(effect);
  }

  @override
  Future<void> close() async {
    await _controller.close();
    return super.close();
  }
}
```

詳細用法、訂閱、生命週期：[`presentation-layer.md`](../architecture/presentation-layer.md#sideeffect-訂閱)。

### ❌ 用 `Cubit`

```dart
class LoginCubit extends Cubit<LoginState> {
  LoginCubit({required this.loginUseCase}) : super(const LoginInitial());
  final LoginUseCase loginUseCase;

  Future<void> login(String email, String password) async {
    emit(const LoginLoading());
    // ... fold result
  }
}
```

**為什麼錯**：Cubit 沒有 Event 概念，使用者意圖不被記錄；BDD 測試無法用 `Given...When LoginSubmitted...Then...` 的天然對齊；多步驟 flow 中間 state 容易失控。
詳見 [ADR-001](../adr/001-bloc-not-cubit.md)。

### ❌ BLoC 直連 Repository

```dart
@injectable
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  LoginBloc({required AuthRepository repository})       // ❌
      : _repository = repository,
        super(const LoginInitial());

  final AuthRepository _repository;
}
```

**為什麼錯**：跳過 UseCase 層；UseCase 是業務邏輯封裝點（驗證、組合多 repository、補資料），跳過去後業務邏輯散在 BLoC handler 各處。

---

## 業務 outcome navigation

業務結果（成功 / 失敗）後的 navigation **必走 SideEffect**：

```dart
// ✅ BLoC 在 fold 後 emit SideEffect
result.fold(
  (failure) => emit(LoginFailure(failure: failure)),
  (user) {
    emit(LoginSuccess(user: user));
    emitSideEffect(const LoginNavigateToHome());     // ✅
  },
);
```

```dart
// ❌ Page 在 onPressed 內直接 navigate
onPressed: () {
  context.read<LoginBloc>().add(LoginSubmitted(email: e, password: p));
  context.go('/home');                              // ❌ outcome 還沒回來
}
```

詳見 [`patterns/navigation.md`](./navigation.md)。

---

## State flag 反 pattern

把一次性事件當 state flag 是常見錯誤：

```dart
// ❌ shouldNavigate 是一次性事件不該綁 state
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState({
    @Default(false) bool shouldNavigate,          // ❌
    User? user,
  }) = _LoginState;
}
```

**為什麼錯**：state 是「持久狀態」，flag 是「一次性事件」；用 state 表達後，需要再 emit `shouldNavigate: false` 重置（醜），而且 rebuild 時可能重複觸發 navigation。

正確做法：用 SideEffect。

---

## Related

- [`architecture/presentation-layer.md`](../architecture/presentation-layer.md) — Page 怎麼消費 State / SideEffect
- [`navigation.md`](./navigation.md) — 業務 navigation 走 SideEffect
- [`error-handling.md`](./error-handling.md) — `Either<AppFailure, T>` 在 BLoC handler 的處理
- [`code-generation.md`](./code-generation.md) — `freezed` codegen 規範
- [ADR-001 ~ 006](../adr/) — BLoC 三件套相關決策
