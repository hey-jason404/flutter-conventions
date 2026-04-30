# Presentation Layer

> **TL;DR**：Page = wire BLoC 跟 UI；Widget 拆成獨立 class（**禁** `_buildXxx()`）；State 用 `BlocBuilder` / `BlocSelector` 消費；SideEffect 用 `BlocListener` 訂閱；Page 不直接 catch exception。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | Widget 拆分用獨立 class，禁 `_buildXxx()` private method | [ADR-010](../adr/010-no-build-prefix-private-method.md) |
| 2 | Page 不直接 catch exception | — |
| 3 | Page 不直接 import data 層 | — |
| 4 | BLoC 只依賴 domain UseCase | — |
| 5 | 一個 page 一個 sub-folder + page-level BLoC | — |
| 6 | 業務 outcome 走 SideEffect、純 UI 走 callback | [ADR-005](../adr/005-imperative-actions-as-side-effect.md) |

---

## Page 結構

每個 page 有自己的目錄：

```text
presentation/login_page/
├── login_page.dart                # 必有
├── bloc/                          # 有狀態才建
│   ├── login_bloc.dart
│   ├── login_event.dart
│   └── login_state.dart
└── widgets/                       # 拆出子 widget 才建
    └── login_form_widget.dart
```

### 命名規則

- Page class：`{PageName}Page`，如 `LoginPage`、`ProfileEditPage`
- Page-level BLoC：`{PageName}Bloc`，**不是** `{Feature}Bloc`
- Widget：`{Name}Widget`，如 `LoginFormWidget`

```dart
// ✅ Page-level BLoC
class LoginBloc extends Bloc<LoginEvent, LoginState> { ... }

// ❌ Feature-level BLoC
class AuthBloc extends Bloc<AuthEvent, AuthState> { ... }
```

**為什麼錯**：feature-level BLoC 跨 page 共用 state，rebuild 範圍失控、test 要 mock 整個 feature 的所有事件。一頁一 BLoC 才能控制 scope。

→ 詳見 [`folder-structure.md`](./folder-structure.md)

### Page 對外 routing contract type

當 page 是被多個 flow / feature 觸發的服務型 page（驗證 / 選取 / 確認類），page 的入口 params 與出口 result type 是 page 的對外契約，**與 page 同居於 page 目錄**：

```text
features/{feature}/presentation/{page_name}/
├── {page_name}_page.dart
├── {page_name}_params.dart        # 入口 params（caller 帶進來）
├── {page_name}_result.dart        # 出口 result（page 回給 caller）
├── bloc/
└── widgets/
```

判斷準則：「**換 router 框架，這個 type 還在嗎？**」

| Type | 用途 | 住哪 |
|---|---|---|
| `{Page}Params` | 只為了把資料送進 page 存在 | page 同目錄 |
| `{Page}Result` | 只為了把結果送回 caller 存在 | page 同目錄 |
| 業務分流 enum（譬如 `VerificationScenario`，UseCase 內 send / verify 邏輯分流用） | 驅動業務邏輯 | `domain/enums/` |
| 業務 entity（譬如 `VerificationToken`，驗證成功的業務資料） | 驅動業務邏輯 | `domain/entities/` |

具體例：service feature `VerificationPage`：

```text
features/verification/presentation/verification_page/
├── verification_page.dart
├── verification_params.dart       # ✅ 對外 routing contract
├── verification_result.dart       # ✅ 對外 routing contract
├── bloc/                          # ❌ 不對外
└── widgets/
```

Page + 其 routing contract types 跨 feature **可被 import**（規範視為 page 的公開介面）；BLoC / state / event / sideEffect **不可跨 feature import**。

→ 詳見 [ADR-014](../adr/014-features-cross-import-rules.md)

---

## Page 內部結構

```dart
// ✅ Page 是 StatefulWidget（需要 SideEffect 訂閱）
class LoginPage extends StatefulWidget {
  const LoginPage({super.key});

  @override
  State<LoginPage> createState() => _LoginPageState();
}

class _LoginPageState extends State<LoginPage> {
  late final LoginBloc _bloc;
  StreamSubscription<LoginSideEffect>? _sideEffectSub;

  @override
  void initState() {
    super.initState();
    _bloc = getIt<LoginBloc>();
    _sideEffectSub = _bloc.sideEffects.listen(_onSideEffect);
  }

  @override
  void dispose() {
    _sideEffectSub?.cancel();
    _bloc.close();
    super.dispose();
  }

  void _onSideEffect(LoginSideEffect effect) {
    switch (effect) {
      case LoginNavigateToHome():
        context.go('/home');
      case LoginShowError(:final message):
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(message)),
        );
    }
  }

  @override
  Widget build(BuildContext context) {
    return BlocProvider.value(
      value: _bloc,
      child: const _LoginScaffold(),
    );
  }
}

class _LoginScaffold extends StatelessWidget {
  const _LoginScaffold();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: const _LoginBody(),
    );
  }
}

class _LoginBody extends StatelessWidget {
  const _LoginBody();

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<LoginBloc, LoginState>(
      builder: (context, state) {
        return switch (state) {
          LoginInitial() => const LoginFormWidget(),
          LoginLoading() => const Center(child: CircularProgressIndicator()),
          LoginSuccess() => const SizedBox.shrink(),
          LoginFailure(:final failure) => _LoginErrorWidget(failure: failure),
        };
      },
    );
  }
}
```

---

## Widget 拆分

### 規則

- 一個 `build()` method 不超過 ~50 行
- 超過抽**獨立 class**（`StatelessWidget` / `StatefulWidget`）
- **禁用 `_buildXxx()` private method**
- 抽出的 widget：page 私用放同檔尾（`class _LoginBody`）；feature 內共用放 `widgets/`

### ❌ 用 `_buildXxx()` private method

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: _buildAppBar(),                   // ❌
      body: Column(
        children: [
          _buildEmailField(),                    // ❌
          _buildPasswordField(),                 // ❌
          _buildSubmitButton(),                  // ❌
        ],
      ),
    );
  }

  AppBar _buildAppBar() => AppBar(title: const Text('Login'));
  Widget _buildEmailField() { ... }
  Widget _buildPasswordField() { ... }
  Widget _buildSubmitButton() { ... }
}
```

**為什麼錯**：

1. **失去 const 優化** —— function 回傳的 widget 無法 const 化、每次 build 重建
2. **DevTools widget tree 看不到** —— 顯示為「一個大 build」，無法 inspect 子部分
3. **Rebuild 範圍失控** —— 整個 build() 重跑，獨立 widget 才能被 framework 跳過
4. **無法獨立測試** —— widget test 必須建整個 page

### ✅ 用獨立 class

```dart
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Login')),
      body: const _LoginForm(),
    );
  }
}

class _LoginForm extends StatelessWidget {
  const _LoginForm();

  @override
  Widget build(BuildContext context) {
    return Column(
      children: const [
        _EmailField(),
        _PasswordField(),
        _SubmitButton(),
      ],
    );
  }
}

class _EmailField extends StatelessWidget {
  const _EmailField();
  @override
  Widget build(BuildContext context) => const TextField(decoration: InputDecoration(hintText: 'Email'));
}

class _PasswordField extends StatelessWidget {
  const _PasswordField();
  @override
  Widget build(BuildContext context) => const TextField(decoration: InputDecoration(hintText: 'Password'), obscureText: true);
}

class _SubmitButton extends StatelessWidget {
  const _SubmitButton();
  @override
  Widget build(BuildContext context) => ElevatedButton(onPressed: () {}, child: const Text('Submit'));
}
```

→ 詳見 [ADR-010](../adr/010-no-build-prefix-private-method.md)

---

## 狀態消費

### `BlocBuilder`：消費整個 State

```dart
BlocBuilder<LoginBloc, LoginState>(
  builder: (context, state) {
    return switch (state) {
      LoginInitial() => const _LoginForm(),
      LoginLoading() => const Center(child: CircularProgressIndicator()),
      LoginSuccess() => const _SuccessWidget(),
      LoginFailure(:final failure) => _ErrorWidget(failure: failure),
    };
  },
);
```

### `BlocSelector`：只看 State 的某個 slice

當 widget 只關心 state 的一小部分，用 `BlocSelector` 避免 over-rebuild：

```dart
BlocSelector<LoginBloc, LoginState, bool>(
  selector: (state) => state is LoginLoading,
  builder: (context, isLoading) {
    return ElevatedButton(
      onPressed: isLoading ? null : () => context.read<LoginBloc>().add(LoginSubmitted(...)),
      child: isLoading ? const CircularProgressIndicator() : const Text('Submit'),
    );
  },
);
```

### `BlocListener`：State change 觸發副作用（謹慎使用）

⚠️ **大多數情況用 SideEffect，不要用 `BlocListener` 監聽 state**。

```dart
// ❌ 用 BlocListener 監聽 state 跳 navigation
BlocListener<LoginBloc, LoginState>(
  listener: (context, state) {
    if (state is LoginSuccess) {
      context.go('/home');                     // ❌
    }
  },
  child: const _LoginForm(),
)
```

**為什麼錯**：state 是「BLoC 當前在哪」，不是「剛剛發生什麼」；登入成功是一次性事件，不該綁 state。
→ 用 SideEffect 詳見 [`patterns/state-management.md`](../patterns/state-management.md)、[`patterns/navigation.md`](../patterns/navigation.md)

---

## SideEffect 訂閱

Page 在 `initState` 訂閱、`dispose` 取消：

```dart
class _LoginPageState extends State<LoginPage> {
  late final LoginBloc _bloc;
  StreamSubscription<LoginSideEffect>? _sideEffectSub;

  @override
  void initState() {
    super.initState();
    _bloc = getIt<LoginBloc>();
    _sideEffectSub = _bloc.sideEffects.listen(_onSideEffect);
  }

  @override
  void dispose() {
    _sideEffectSub?.cancel();
    _bloc.close();
    super.dispose();
  }

  void _onSideEffect(LoginSideEffect effect) {
    switch (effect) {
      case LoginNavigateToHome():
        context.go('/home');
      case LoginShowError(:final message):
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(message)));
      case LoginConfirmLogout():
        showDialog(context: context, builder: (_) => const _ConfirmLogoutDialog());
    }
  }
  // ... build()
}
```

---

## 錯誤處理邊界

### Page **不直接 catch exception**

```dart
// ❌ Page 內 try-catch
@override
Widget build(BuildContext context) {
  return ElevatedButton(
    onPressed: () async {
      try {
        await _useCase(input: ...);
      } catch (e) {
        ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('$e')));
      }
    },
    child: const Text('Submit'),
  );
}
```

**為什麼錯**：Page 應該是 dumb wire UI ↔ BLoC，業務錯誤應透過 BLoC State / SideEffect 表達。

### 正確做法：透過 State 衍生 UI

```dart
// ✅ Page 看 State 渲染對應 UI
BlocBuilder<LoginBloc, LoginState>(
  builder: (context, state) {
    return switch (state) {
      LoginFailure(:final failure) => _ErrorWidget(failure: failure),
      // ... other states
    };
  },
);

// ✅ Error 詳細訊息透過 SideEffect emit snackbar
void _onSideEffect(LoginSideEffect effect) {
  if (effect is LoginShowError) {
    ScaffoldMessenger.of(context).showSnackBar(
      SnackBar(content: Text(effect.message)),
    );
  }
}
```

---

## Loading / Error / Empty UI

從 State 衍生：

```dart
return switch (state) {
  LoginInitial() => const _InitialView(),
  LoginLoading() => const Center(child: CircularProgressIndicator()),
  LoginSuccess(:final user) => _SuccessView(user: user),
  LoginFailure(:final failure) => _ErrorView(failure: failure),
};
```

### Empty state（資料正常但結果為空）

State 設計時就要區分 `Loaded(items: [])` vs `Empty()`：

```dart
@freezed
sealed class ProductListState with _$ProductListState {
  const factory ProductListState.initial() = ProductListInitial;
  const factory ProductListState.loading() = ProductListLoading;
  const factory ProductListState.loaded({required List<Product> items}) = ProductListLoaded;
  const factory ProductListState.empty() = ProductListEmpty;
  const factory ProductListState.failure({required AppFailure failure}) = ProductListFailure;
}
```

---

## 常見錯誤

```dart
// ❌ Page 直接 import data 層
import '../../../data/datasources/remote/auth_remote_datasource.dart';

class LoginPage extends StatelessWidget { ... }
```
**為什麼錯**：跨層違規。Page 只能 reach 到 BLoC，不能跨過 Domain / Data。

```dart
// ❌ Page 自己呼 UseCase（不透過 BLoC）
class LoginPage extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ElevatedButton(
      onPressed: () => getIt<LoginUseCase>()(input: ...),    // ❌
      child: const Text('Login'),
    );
  }
}
```
**為什麼錯**：UI 直接呼 UseCase 跳過 BLoC，狀態管理失控；BLoC 是 UI ↔ business 的中介，不能繞。

```dart
// ❌ State flag 用於一次性事件
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState({
    @Default(false) bool shouldNavigate,        // ❌
    User? user,
  }) = _LoginState;
}
```
**為什麼錯**：navigation 是一次性事件、不該綁在 state；應該走 SideEffect。

---

## Related

- [`folder-structure.md`](./folder-structure.md) — Page / BLoC / Widget 目錄結構
- [`patterns/state-management.md`](../patterns/state-management.md) — BLoC / Event / State / SideEffect
- [`patterns/navigation.md`](../patterns/navigation.md) — 業務 outcome navigation 走 SideEffect
- [ADR-010](../adr/010-no-build-prefix-private-method.md)、[ADR-005](../adr/005-imperative-actions-as-side-effect.md)、[ADR-014](../adr/014-features-cross-import-rules.md)
