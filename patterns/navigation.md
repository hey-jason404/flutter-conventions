# Navigation

> **TL;DR**：業務 outcome（成功 / 失敗 / 表單送出後）走 SideEffect；純 UI navigation（開設定、上一頁）直接 `context.go()` / `context.pop()` 在 callback。用 `go_router`。

---

## 不可妥協

| # | 規則 | ADR |
|---|---|---|
| 1 | 業務 outcome navigation 走 SideEffect，禁在 `BlocListener` 監聽 state 跳 | [ADR-005](../adr/005-imperative-actions-as-side-effect.md) |
| 2 | 純 UI navigation 直接在 callback 用 `context.go()` / `context.pop()` | — |
| 3 | 路由集中於統一 router config | — |
| 4 | 禁用 state flag（`bool shouldNavigate`）觸發 navigation | — |
| 5 | URL hardcoded 路徑禁出現散落 widget code 內，集中於 routes constant | — |

---

## 兩種 navigation：判斷依據

| 情境 | 判斷 | 怎麼做 |
|---|---|---|
| **業務 outcome** | 結果有 outcome（成功 / 失敗）才跳 | BLoC emit SideEffect → Page 訂閱跳 |
| **純 UI navigation** | 不依賴業務結果（開設定、上一頁、開分頁） | `context.go()` / `context.pop()` 直接在 callback |

判斷問題：「**這次跳轉需不需要等某個 async 結果？**」
- 需要 → SideEffect
- 不需要 → 直接 navigation

---

## 業務 outcome navigation：走 SideEffect

### ✅ 完整流程

**BLoC 在 fold 後 emit SideEffect：**

```dart
@injectable
class LoginBloc extends Bloc<LoginEvent, LoginState>
    with BlocSideEffectsMixin<LoginSideEffect> {
  // ...
  Future<void> _onLoginSubmitted(
    LoginSubmitted event,
    Emitter<LoginState> emit,
  ) async {
    emit(const LoginLoading());
    final result = await _loginUseCase(input: LoginInput(...));
    result.fold(
      (failure) {
        emit(LoginFailure(failure: failure));
        emitSideEffect(LoginShowError(message: failure.message));
      },
      (user) {
        emit(LoginSuccess(user: user));
        emitSideEffect(const LoginNavigateToHome());     // ✅ outcome 後跳
      },
    );
  }
}
```

**Page 訂閱 SideEffect：**

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
        context.go(Routes.home);                          // ✅ 在 SideEffect handler 跳
      case LoginShowError(:final message):
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(message)),
        );
    }
  }

  @override
  Widget build(BuildContext context) {
    return BlocProvider.value(value: _bloc, child: const _LoginScaffold());
  }
}
```

---

## 純 UI navigation：直接呼

不依賴業務結果的，在 callback 直接 navigate：

```dart
// ✅ 開設定（純 UI，無 outcome）
TextButton(
  onPressed: () => context.go(Routes.settings),
  child: const Text('Settings'),
)

// ✅ 上一頁
IconButton(
  icon: const Icon(Icons.arrow_back),
  onPressed: () => context.pop(),
)

// ✅ 開 modal / dialog（純 UI 操作）
TextButton(
  onPressed: () => showDialog(
    context: context,
    builder: (_) => const _AboutDialog(),
  ),
  child: const Text('About'),
)
```

---

## 反 pattern

### ❌ 在 `onPressed` 內 add Event 又 navigate

```dart
ElevatedButton(
  onPressed: () {
    context.read<LoginBloc>().add(LoginSubmitted(email: e, password: p));
    context.go(Routes.home);                  // ❌ outcome 還沒回來就跳
  },
  child: const Text('Submit'),
)
```

**為什麼錯**：login 是 async；`context.go(Routes.home)` 在 add 之後立刻執行，**根本不等 outcome**。如果 login 失敗，使用者已經被導到 home。

### ❌ 用 `BlocListener` 監聽 state 跳

```dart
BlocListener<LoginBloc, LoginState>(
  listener: (context, state) {
    if (state is LoginSuccess) {
      context.go(Routes.home);                // ❌
    }
  },
  child: const _LoginForm(),
)
```

**為什麼錯**：

1. State 是「BLoC 當前在哪」，不是「剛剛發生什麼事件」；登入成功是一次性事件，不該綁 state
2. State 重新進入 LoginSuccess（譬如熱重啟、儲存的 BLoC 重建）會**重複觸發 navigation**
3. SideEffect 是專為一次性事件設計的機制；繞過去用 state 是 misuse

詳見 [ADR-005](../adr/005-imperative-actions-as-side-effect.md)。

### ❌ State flag 觸發 navigation

```dart
@freezed
sealed class LoginState with _$LoginState {
  const factory LoginState({
    @Default(false) bool shouldNavigate,      // ❌
    User? user,
  }) = _LoginState;
}

// page
BlocListener<LoginBloc, LoginState>(
  listener: (context, state) {
    if (state.shouldNavigate) {
      context.go(Routes.home);
      context.read<LoginBloc>().add(LoginNavigationConsumed());  // 還要 reset flag
    }
  },
  child: const _LoginForm(),
)
```

**為什麼錯**：把一次性事件當持久 state，需要再 reset flag；不直觀、容易出 bug。SideEffect 才是設計給這場景。

---

## `go_router` 規則

### Route 集中

```dart
// ✅ routes/app_router.dart
abstract final class Routes {
  static const String login = '/login';
  static const String home = '/home';
  static const String profile = '/profile';
  static const String productDetail = '/product/:id';
}

final appRouter = GoRouter(
  initialLocation: Routes.login,
  routes: [
    GoRoute(
      path: Routes.login,
      builder: (context, state) => const LoginPage(),
    ),
    GoRoute(
      path: Routes.home,
      builder: (context, state) => const HomePage(),
    ),
    GoRoute(
      path: Routes.productDetail,
      builder: (context, state) {
        final id = state.pathParameters['id']!;
        return ProductDetailPage(productId: id);
      },
    ),
  ],
);
```

```dart
// ❌ hardcoded 路徑散落 widget
TextButton(
  onPressed: () => context.go('/settings'),    // ❌ 字面值
  child: const Text('Settings'),
)
```

**為什麼錯**：rename 路徑時 grep 不全；typo 不會編譯期擋。

### 參數傳遞

| 場景 | 方式 |
|---|---|
| Resource ID（產品 ID、訂單號） | path parameter — `/product/:id` |
| Filter / pagination | query parameter — `/products?category=foo&page=2` |
| 大型物件（已 load 的 entity） | `extra` —— 但**不要過度依賴**（refresh / deep link 拿不到） |
| 表單預填值 | path / query 適合的；多欄位用 query |

```dart
// ✅ path parameter
context.go('/product/${product.id}');

// ✅ query parameter
context.go('/products?category=$category&page=$page');

// ✅ extra（謹慎）
context.go('/product-detail', extra: product);

// 對應 GoRoute
GoRoute(
  path: '/product-detail',
  builder: (context, state) {
    final product = state.extra as Product;
    return ProductDetailPage(product: product);
  },
);
```

---

## Service feature handoff

跨 feature 跳到「服務型 page」（驗證 / 選取 / 確認類）並等回傳結果，走 `app/` 內的 router facade，**不直接 import 對方 page widget**。

### 規則

- caller import 對方的 **domain entity / enum** + 對方的 **page-level routing contract type**（params / result，與 page 同居）
- caller **不** import 對方的 page widget、BLoC、state、event
- navigation 透過 `app/` 內 type-safe facade 觸達；facade 內封裝 `go_router` 細節
- 路徑常數仍集中於 `Routes`

### ✅ Caller 端

```dart
// features/profile/.../change_email_bloc.dart
import 'package:.../app/router/app_router.dart';                                                // ✅ from app
import 'package:.../features/verification/presentation/verification_page/verification_params.dart'; // ✅ page contract
import 'package:.../features/verification/presentation/verification_page/verification_result.dart'; // ✅ page contract
import 'package:.../features/verification/domain/enums/verification_scenario.dart';            // ✅ domain enum

@injectable
class ChangeEmailBloc extends Bloc<ChangeEmailEvent, ChangeEmailState> {
  ChangeEmailBloc({
    required UpdateEmailUseCase updateEmail,
    required AppRouter router,
  });

  Future<void> _onSubmitRequested(...) async {
    final result = await _router.pushVerification(
      params: VerificationParams(scenario: VerificationScenario.changeEmail),
    );
    if (!result.verified) return;
    // ...
  }
}
```

### ❌ 反 pattern

```dart
// ❌ caller 直接 import 對方 page widget 並 push
import 'package:.../features/verification/presentation/verification_page/verification_page.dart';

final result = await Navigator.of(context).push(
  MaterialPageRoute(builder: (_) => VerificationPage(params: ...)),
);
```

**為什麼錯**：繞過 `go_router`，deep link / route metadata 失效；caller feature 跟對方 page 的內部建構式綁死。

→ 詳見 [ADR-014](../adr/014-features-cross-import-rules.md)

## SideEffect 命名

依「動作類型 + 對象」命名，描述明確：

```dart
// ✅ 命名清楚
sealed class LoginSideEffect {}

final class LoginNavigateToHome extends LoginSideEffect {
  const LoginNavigateToHome();
}

final class LoginNavigateToOnboarding extends LoginSideEffect {
  const LoginNavigateToOnboarding();
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
// ❌ 命名籠統
final class LoginNavigate extends LoginSideEffect {       // 跳哪？
  const LoginNavigate();
}

final class LoginAction extends LoginSideEffect {         // 什麼 action？
  const LoginAction(this.type);
  final String type;
}
```

---

## Deep link / 多重入口場景

當 page 可從多個 entry 進入（deep link、推播、normal navigation），**不在 BLoC 假設來源**；BLoC 只表達「當前頁要去哪」，**怎麼去**由 router config 決定：

```dart
// ✅ BLoC 只 emit "需要去 home"，router 決定 stack 該怎麼處理
emitSideEffect(const LoginNavigateToHome());

// router config 內處理 stack 行為（go vs push vs pushReplacement）
```

---

## Related

- [`state-management.md`](./state-management.md) — SideEffect 結構與 emit 機制
- [`architecture/presentation-layer.md`](../architecture/presentation-layer.md) — Page 訂閱 SideEffect 流程
- [`packages.md`](../packages.md) — `go_router` 在白名單
- [ADR-005](../adr/005-imperative-actions-as-side-effect.md)、[ADR-006](../adr/006-side-effect-plain-sealed-class.md)、[ADR-014](../adr/014-features-cross-import-rules.md)
